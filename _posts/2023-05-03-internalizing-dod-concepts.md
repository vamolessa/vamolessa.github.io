---
title: "internalizing data-oriented concepts"
---

When porting [my code editor](https://github.com/vamolessa/pepper) to C,
I was trying to shape some core code that dealt with buffer editing and its undo system.
I noticed then that the api I was making (like the original version), dealt with one buffer edit at a time.
Even though the editor supports multiple cursors which make multiple edits at once.

It took me some time to reach a nice api that accepted batch editing.
Only as I was completing the refactor, I was able to see exactly what codepaths were unnecessarily repeated
for each edit:

The flow first starts at the buffer view which contains cursors info for a certain buffer:

```c
// before
void
view_replace_text(struct View* view, struct Text text) {
    for (i32 cursor_i = (i32)view->cursors_len - 1; cursor_i >= 0; cursor_i--) {
        // edit buffer directly multiple times

        struct Range range = cursor_range(view->cursors[cursor_i]);
        buffer_replace_text(view->buffer, range, text);
    }
}

// after
void
view_replace_text(struct View* view, struct Text text) {
    struct Edit edits[LEN(view->cursors)];
    struct Edit* edits_at = edits;
    for (i32 i = (i32)view->cursors_len - 1; i >= 0; i--) {
        // gather edits for each cursor in reverse order

        *edits_at++ = (struct Edit){
            .range = cursor_range(view->cursors[i]),
            .text = text,
        };
    }

    // process all edits to buffer in batch
    buffer_edit(view->buffer, edits, view->cursors_len);
}
```

> NOTE: both loops iterate in reverse over the (sorted) cursor array because
> we don't want earlier edits to influence later edits!

Now, it may seem that we are only doing more work, but it makes sense when we see
how the buffer implementation changed:

```c
// before
static void
buffer_replace_text_no_history(struct Buffer* buffer, struct Range range, struct Text text) {
    // we split the edit function in two because we reuse it by the undo system
    // but then we don't want to record undo edits while navigating the undo stack!
    // which is why this function does not records undo edits

    u32 range_len = range.end - range.start;
    u32 gap_len = buffer->gap_end - buffer->gap_start;
    if (gap_len + range_len < text.len) {
        // NOTE: gap too small
        buffer_ensure_gap_len(buffer, gap_len + text.len - range_len);
    }

    buffer_move_gap_to(buffer, range.end);

    // NOTE: deletion
    buffer->gap_start -= range_len;

    // NOTE: insertion
    mem_copy(buffer->content_arena.mem + buffer->gap_start, text.ptr, text.len);
    buffer->gap_start += text.len;
}

void
buffer_replace_text(struct Buffer* buffer, struct Range range, struct Text text) {
    // NOTE: since we're using a gap buffer,
    // the text in range could be split by the gap
    // which is why `buffer_text_in_range` returns two `struct Text`
    struct Text deleted_texts[2];
    buffer_text_in_range(buffer, range, deleted_texts);
    for (i32 i = LEN(deleted_texts) - 1; i >= 0; i--) {
        struct Text deleted_text = deleted_texts[i];
        if (deleted_text.len) {
            // if we're deleting, push a delete change to the undo history

            u32 pos = buffer_pos_from_ptr(buffer, deleted_text.ptr);
            edit_history_push(&buffer->history, /* is_insert */ false, pos, deleted_text);
        }
    }

    buffer_replace_text_no_history(buffer, range, text);

    if (text.len) {
        edit_history_push(&buffer->history, /* is_insert */ true, range.start, text);
    }
}

// after
static void
buffer_edit_no_history(struct Buffer* buffer, const struct Edit edits[], u32 edits_len) {
    u32 total_delete_len = 0;
    u32 total_insert_len = 0;
    for (u32 i = 0; i < edits_len; i++) {
        struct Edit edit = edits[i];
        total_delete_len += edit.range.end - edit.range.start;
        total_insert_len += edit.text.len;
    }

    // note how we now can hoist the gap resizing out of the editing loop!
    u32 gap_len = buffer->gap_end - buffer->gap_start;
    if (gap_len + total_delete_len < total_insert_len) {
        // NOTE: gap too small
        buffer_ensure_gap_len(buffer, gap_len + total_insert_len - total_delete_len);
    }

    for (u32 i = 0; i < edits_len; i++) {
        struct Edit edit = edits[i];
        buffer_move_gap_to(buffer, edit.range.end);

        // NOTE: deletion
        u32 delete_len = edit.range.end - edit.range.start;
        buffer->gap_start -= delete_len;

        // NOTE: insertion
        mem_copy(buffer->content_arena.mem + buffer->gap_start, edit.text.ptr, edit.text.len);
        buffer->gap_start += edit.text.len;
    }
}

void
buffer_edit(struct Buffer* buffer, const struct Edit edits[], u32 edits_len) {
    // since edits are alreay batched, we can just push them down to the undo system

    edit_history_record(buffer, edits, edits_len);
    buffer_edit_no_history(buffer, edits, edits_len);
}
```

With this change we only need to call the undo system once (`edit_history_record` instead of several `edit_history_push`).
And on top of that, we were able to hoist gap resizing out of the batch loop.
Previouly we were checking and possibly resizing the gap multiple times for a batch of multiple cursors edit, but not anymore.

And now the final piece: the undo system changes:

```c
// before
void
edit_history_push(struct EditHistory* history, b32 is_insert, u32 pos, struct Text text) {
    edit_history_crate_new_group_if_not_recording(history);

    // `history->current` is the current undo edit group
    // containing all the edits applied with a single undo operation
    history->current->edits_len += 1;

    const char* edit_text = alloc_uninit(&history->texts_arena, char, text.len);
    mem_copy(edit_text, SLICE_MEM(text.ptr, text.len));

    struct HistoryEdit* edit = alloc_uninit(&history->edits_arena, struct HistoryEdit, 1);
    *edit = (struct HistoryEdit){
        .is_insert = is_insert,
        .pos = pos,
        .len = text.len,
        .text = edit_text,
    };
}

// after
void
edit_history_record(struct Buffer* buffer, const struct Edit edits[], u32 edits_len) {
    struct EditHistory* history = &buffer->history;

    // also hoisted out of the main edit loop
    edit_history_crate_new_group_if_not_recording(history);

    history->current->edits_len += edits_len;
    struct HistoryEdit* write_at = alloc_uninit(&history->edits_arena, struct HistoryEdit, edits_len);

    struct Arena* texts_arena = &history->texts_arena;
    for (i32 edit_i = (i32)edits_len - 1; edit_i >= 0; edit_i--) {
        struct Edit edit = edits[edit_i];
        struct HistoryEdit history_edit = {
            .pos = edit.range.start,
            .texts_start = (u32)texts_arena->len,
            .insert_len = edit.text.len,
        };

        struct Text delete_texts[2];
        buffer_text_in_range(buffer, edit.range, delete_texts);
        for (i32 i = LEN(delete_texts) - 1; i >= 0; i--) {
            struct Text deleted = delete_texts[i];
            arena_push(texts_arena, SLICE_MEM(deleted.ptr, deleted.len));
            history_edit.delete_len += deleted.len;
        }
        arena_push(texts_arena, SLICE_MEM(edit.text.ptr, edit.text.len));

        *write_at++ = history_edit;
    }
}
```

> NOTE: with the refactor, I took the opportunity to rework the undo edit storage.
> It now only stores the edit position, the deleted text length, the inserted text length
> and an offset into the undo's texts_arena.
> The offset can be used to obtain the deleted text content which is then followed by the insert text content.

Now we don't need to check, for every cursor, if we're not already recording undo edits.
As a nice bonus (due to both operating in batches and the HistoryEdit's storage change),
we can now guarantee that the number of edits recorded is equal to the number of edits made.

Before we could have at most 3 calls to `edit_history_push` for every `buffer_replace_text` call.
This will make it much easier to implement undo edit merge later on (which saves memory,
make less undo edits, and lets us to nicely position cursors for each undo change when navigating history).

Finally, when undoing/redoing, we can now just pass the undo edit batch directly to `buffer_edit`:

```c
// before
const struct Edit*
buffer_undo(struct Buffer* buffer, struct Arena* arena) {
    const struct Edit* edits = edit_history_undo(&buffer->history, arena);
    const struct Edit* edits_end = arena_top(arena, alignof(struct Edit));
    for (const struct Edit* at = edits; at != edits_end; at++) {
        buffer_replace_text_no_history(buffer, at->range, at->text);
    }
    return edits;
}

// after 
const struct Edit*
buffer_undo(struct Buffer* buffer, struct Arena* arena) {
    const struct Edit* edits = edit_history_undo(&buffer->history, arena);
    u32 edits_len = (u32)arena_len_from(arena, edits);
    // no loop, just directly process the undo edit batch
    buffer_edit_no_history(buffer, edits, edits_len);
    return edits;
}
```

When coding the original version of the editor, I had to make a similar change to the event system.
I had changed from single buffer edit messages to messages of multiple buffer edits.
This enabled me to deal with a accidentally quadratic situation which then let me efficiently process
several hundreds of cursors (it was some log massaging I had to do which exposed the problem).

Writing apis that work in batches makes it much easier to spot workloads that only need to be processed once;
which would otherwise be called inside a loop hidden behind several function calls.

I feel like I've internalized data-oriented concepts ("where there's one, there's many") a little bit more.
