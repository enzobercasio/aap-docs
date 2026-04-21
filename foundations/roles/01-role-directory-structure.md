# Role Directory Structure

Imagine you are walking into a massive industrial kitchen. If every chef decided to store the salt in a different place—one in a drawer, one on a high shelf, and one in their pocket—the kitchen would collapse into chaos the moment a rush hit. To function at scale, the kitchen needs a "standard operating procedure": knives go here, spices go there, and the cold storage is always in the back.

In Ansible, **Roles** are that industrial kitchen. They provide a rigid, predictable structure that allows you to break a complex playbook into small, manageable, and reusable pieces. When you follow the standard directory layout, you aren't just organizing files; you are using a "shared language" that every other Ansible architect in the world understands instantly.

## The Anatomy of a Role

When we talk about a role, we are talking about a specific directory tree. Ansible is "opinionated," meaning it expects certain things to be in certain places. If you follow its rules, Ansible rewards you by automatically loading your tasks, variables, and handlers without you having to write a single line of code to "import" them.

Let’s look at a concrete example. Suppose we are building a role called `apache` to manage a web server. Here is what a professional-grade role looks like:

```text
apache/
├── defaults/
│   └── main.yml         # Public variables (the "knobs" for the user)
├── files/
│   └── index.html       # Static files to be copied
├── handlers/
│   └── main.yml         # Service restart logic
├── meta/
│   └── main.yml         # Role metadata (author, license, dependencies)
├── tasks/
│   └── main.yml         # The "meat": the list of actions to take
├── templates/
│   └── vhost.conf.j2    # Dynamic Jinja2 templates
├── tests/
│   ├── inventory
│   └── test.yml         # Local testing files
└── vars/
    └── main.yml         # Internal constants (not meant to be changed)
```

### The Rule of `main.yml`

You’ll notice that almost every directory contains a file named `main.yml`. This is the **Entry Point**.

When you tell a playbook to use the `apache` role, Ansible doesn't just look at the folder; it immediately goes hunting for `apache/tasks/main.yml`. If that file exists, it executes it. If it doesn't find it, the role fails. The same applies to `handlers`, `defaults`, and `vars`. Ansible specifically looks for `main.yml` to know what to "auto-load" into the execution environment.

## Breaking Down the Directories

To be an effective advisor, you need to know exactly what belongs in each "drawer" of the role kitchen. Let's walk through them one by one.

### tasks/

This is the heart of the role. Every task you would normally put in a playbook goes here. 

- **What goes here:** Modules like `apt`, `yum`, `service`, `template`, and `copy`.

- **Pro Tip:** As roles grow, `main.yml` can become a "wall of text." Architects often split tasks into separate files like `install.yml` and `configure.yml`, then use `import_tasks: install.yml` inside the `main.yml` to keep things tidy.

### handlers/

Handlers are tasks that only run when "notified" by another task (usually because a change occurred). 

- **What goes here:** Almost exclusively service restarts (e.g., "Restart Apache if the config changed").

- **Why here?** By putting them in their own directory, you keep your main task list clean and focused on the "what," while the handlers handle the "reactions."

### defaults/ vs. vars/

This is the most frequent point of confusion for beginners. 

- **defaults/**: These are the "tunable knobs." If a user wants to change the Apache port from 80 to 8080, they should find that variable here. Variables in `defaults/` have the lowest priority in Ansible, meaning they are very easy for a user to override in a playbook.

- **vars/**: These are "internal constants." If you have a variable that defines the specific path to a config file on Ubuntu vs. RHEL, and you don't want the user to accidentally break the role by changing it, put it here. Variables in `vars/` have a much higher priority and are harder to override.

### files/ vs. templates/

This distinction is critical for performance and logic.

- **files/**: This directory holds **static** content. If you have a corporate logo or a standard `.txt` file that is exactly the same on every single server, put it here. Ansible uses the `copy` module to move these files bit-for-bit.

- **templates/**: This directory holds **dynamic** content. These files usually end in `.j2` because they use the Jinja2 templating engine. If your configuration file needs to include the server's IP address or a custom hostname, it’s a template. Ansible uses the `template` module to "render" these files (replace the variables with real values) before sending them to the target.

### meta/

Think of this as the "Identity Card" of your role.

- **What goes here:** The author's name, the supported operating systems (e.g., "Ubuntu 22.04"), the license (MIT, Apache, etc.), and **dependencies**.

- **Dependencies:** If your `apache` role requires a `firewall` role to be run first, you declare that in `meta/main.yml`. Ansible will ensure the `firewall` role executes before it even touches the `apache` tasks.