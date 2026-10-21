## Step 1
open the directory which you want to create the temporary server in.
- check your ip address by running the following command in the terminal:

    ```bash
        ipconfig
    ```

## Step 2
Right-click on the directory and select "Open in Terminal" (or "Open in Command Prompt" on Windows) and run the following command to create a temporary server:

```bash
    python -m http.server 6789 --bind 0.0.0.0
```

## Step 3
You can now access the temporary server from any device on the same network by entering the following URL in a web browser:

```
    http://<your-ip-address>:6789
```