# a new c memory primitive: mem_write (apr 2023)

it was when i was writing code for hot reloading that i was dealing with path concatenation.
i didn't get it right the first time. there was lots of calls to `mem_copy`
(just `memcpy` but in my style of `count` params first) and it was kinda error prone one could say.

there had to be a better way to code it.

```c
static b32
try_reload_app_data(struct AppData* app, u32 current_dir_path_len, const WCHAR* current_dir_path) {
    const WCHAR app_lib_path[] = L"build/gamelib.dll";
    const WCHAR loaded_suffix[] = L".loaded";

    WCHAR path_buf[MAX_PATH] = {0};
    ASSERT(current_dir_path_len + STR_LEN(app_lib_path) + STR_LEN(loaded_suffix) < LEN(path_buf));
    u32 current_dir_path_size = current_dir_path_len * sizeof(current_dir_path[0]);

    // we first copy current_dir_path into path_buf
    mem_copy(
        current_dir_path_size,
        path_buf,
        current_dir_path
    );
    // then concatenate app_lib_path to it
    mem_copy(
        sizeof(app_lib_path),
        (char*)path_buf + current_dir_path_size,
        app_lib_path
    );
    // given that current_dir_path contains "my_project_path",
    // path_buf should contain "my_project_path/build/gamelib.dll"

    // then copy it all to loaded_path_buf
    WCHAR loaded_path_buf[LEN(path_buf)] = {0};
    mem_copy(
        current_dir_path_size + sizeof(app_lib_path) - 1,
        loaded_path_buf,
        path_buf
    );
    // then concat loaded_suffix to it
    mem_copy(
        sizeof(loaded_suffix),
        (char*)loaded_path_buf + current_dir_path_size + sizeof(app_lib_path) - 1,
        loaded_suffix
    );
    // loaded_path_buf should contain "my_project_path/build/gamelib.dll.loaded"

    // rest of code goes here
    // what it does is copy file at path_buf to loaded_path_buf and then dynamically load that dll
    // done this way so we're free to overwrite the dll at path_buf with a new version for hot reloading
}
```
