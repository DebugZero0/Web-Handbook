# 🚀 VS Code Auto Start Setup (Backend + Frontend)

This guide explains how to configure **VS Code Tasks** so that both the **Backend** and **Frontend** development servers start with a single command or keyboard shortcut.

---

## 📁 Project Structure

Your project should be organized like this:

```text
Project/
│
├── Backend/
│   ├── package.json
│   └── ...
│
├── Frontend/
│   ├── package.json
│   └── ...
│
└── .vscode/
    └── tasks.json
```

---

## Step 1: Create `.vscode/tasks.json`

Inside the project root, create a folder named **.vscode** if it doesn't already exist.

```
Project/
└── .vscode/
    └── tasks.json
```

Paste the following into `tasks.json`:

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Backend",
            "type": "shell",
            "command": "npm run dev",
            "options": {
                "cwd": "${workspaceFolder}/Backend"
            },
            "isBackground": true,
            "presentation": {
                "panel": "dedicated",
                "group": "backend",
                "reveal": "always"
            }
        },
        {
            "label": "Frontend",
            "type": "shell",
            "command": "npm run dev",
            "options": {
                "cwd": "${workspaceFolder}/Frontend"
            },
            "isBackground": true,
            "presentation": {
                "panel": "dedicated",
                "group": "frontend",
                "reveal": "always"
            }
        },
        {
            "label": "Start All",
            "dependsOn": [
                "Backend",
                "Frontend"
            ],
            "dependsOrder": "parallel"
        }
    ]
}
```

---

## Step 2: Run Both Servers

Open the Command Palette:

```
Ctrl + Shift + P
```

Search for:

```
Tasks: Run Task
```

Select:

```
Start All
```

VS Code will automatically:

* Open one terminal for the Backend
* Open another terminal for the Frontend
* Run `npm run dev` in each folder simultaneously

Each server runs independently in its own terminal.

---

## Step 3 (Optional): Add a Keyboard Shortcut

Open the Keyboard Shortcuts JSON:

```
Ctrl + Shift + P
```

Search:

```
Preferences: Open Keyboard Shortcuts (JSON)
```

Add:

```json
[
    {
        "key": "ctrl+alt+w",
        "command": "workbench.action.tasks.runTask",
        "args": "Start All"
    }
]
```

Now pressing:

```
Ctrl + Alt + W
```

will instantly launch both development servers.

---

## Step 4 (Optional): Auto Start on Folder Open

Create:

```
.vscode/settings.json
```

```json
{
    "task.allowAutomaticTasks": "on"
}
```

Modify the `Start All` task:

```json
{
    "label": "Start All",
    "dependsOn": ["Backend", "Frontend"],
    "dependsOrder": "parallel",
    "runOptions": {
        "runOn": "folderOpen"
    }
}
```

Now every time the project is opened in VS Code, both servers will start automatically.

---

## Notes

* This configuration is **project-specific**.
* Every project can have its own `.vscode/tasks.json`.
* The `.vscode` folder can be committed to Git so everyone working on the project gets the same development setup.
* This assumes the project contains:

  * `Backend/package.json`
  * `Frontend/package.json`
* Both projects must define a `"dev"` script in their respective `package.json` files.

---

## Quick Commands

| Action               | Shortcut           |
| -------------------- | ------------------ |
| Open Command Palette | `Ctrl + Shift + P` |
| Run Task             | `Tasks: Run Task`  |
| Start Both Servers   | `Start All`        |
| Custom Shortcut      | `Ctrl + Alt + W`   |

Happy Coding! 🚀
