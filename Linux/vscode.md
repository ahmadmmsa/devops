


vs code install extension "multi-command"


in search "> Open User Settings (JSON)



```json
{
    "multiCommand.commands": [
    {
        "command": "multiCommand.wrapBashBlock",
        "sequence": [
        "editor.action.clipboardCopyAction",
        {
            "command": "editor.action.insertSnippet",
            "args": {
            "snippet": "```bash\n${CLIPBOARD}\n```"
            }
        }
        ]
    }
    ]
}
```



> Open Keyboard Shortcuts (JSON)

```json
[
    {
        "key": "shift+/",
        "command": "multiCommand.wrapBashBlock",
        "when": "editorHasSelection"
    }
]
```