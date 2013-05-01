# Administration Basics

## Command-line arguments


## Configuration files

Instead of entering the settings on the command-line, you can store them in a JSON file and then just provide the path to that file as a command-line argument. As a bonus, the file lets you run multiple databases.

Here's an example configuration file that starts a server with the default settings:

    {
        "interface": ":4984",
        "adminInterface": ":4985",
        "log": ["CRUD", "REST"],
        "databases": {
            "sync_gateway": {
                "server": "http://localhost:8091",
                "bucket": "sync_gateway"
            }
        }
    }

If you want to run multiple databases you can either add more entries to the `databases` property in the config file, or you can put each of them in its own config file (just like above) and list each of the config files on the command line.
