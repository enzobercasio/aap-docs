# defaults vs. vars

If you were designing a high-performance car, you would give the driver a dashboard full of dials and buttons—ways to adjust the seat, the climate, and the radio. However, you wouldn’t put a knob on the dashboard that allows the driver to adjust the fuel-to-air ratio of the engine or the timing of the spark plugs while driving at 70 mph. Those are internal constants, set by the engineers to ensure the machine actually functions.

In the world of Ansible roles, you face a similar design challenge. How do you give your users the flexibility they need to make your role work in their environment without letting them accidentally break the core logic that makes the role reliable?

The answer lies in the "scoping contract" between two specific directories: defaults/ and vars/. Understanding the technical difference in variable precedence is the first step, but the real skill—the architect’s skill—is knowing which variables belong in which bucket to create a role that is both flexible and bulletproof.

The Scoping Contract: Roles as an API

As an Ansible architect, you aren't just writing tasks; you are building a tool for others to use. This "other person" might be a colleague, a different department, or even "Future You" six months from now. To make this tool successful, you need to define a Public API.

In software engineering, an API (Application Programming Interface) defines how one piece of software interacts with another. It says: "Here are the inputs I accept, and here is how you provide them." 

In an Ansible role:

defaults/main.yml is your Public API. These are the settings you expect and want the user to change.

vars/main.yml is your Internal Logic. These are the constants that the role needs to function correctly, which the user should almost never touch.

By strictly separating these two, you provide a clear signal to the user. When they open your role and see a variable in defaults/, they know it’s a "knob" they can turn. If they see a variable in vars/, they should recognize it as part of the "engine" that is best left alone.

defaults/main.yml: The Role’s Public API

The defaults/ directory is where you store the "sane defaults" for your role. Technically, variables defined here have the lowest precedence in all of Ansible (Level 2 out of 22). 

Why is the lowest precedence actually a feature? Because it makes the variable easy to override.

Why Use Defaults?

When you write a role to install a web server, you might assume it should run on port 80. However, a user might need it to run on port 8080 because of a firewall restriction. By putting web_port: 80 in defaults/main.yml, you are saying: "I suggest using port 80, but if you have a better idea, go ahead and tell me."

Because these variables are so easy to override, the user can change them in their playbook, their inventory, or via the command line without having to modify the role's internal code.

Example: A Clean defaults/main.yml

A well-architected defaults/main.yml acts as documentation. It tells the user exactly what they can customize.

---
# defaults/main.yml for nginx_wrapper role

# The port the web server will listen on
nginx_port: 80

# The user the nginx process will run as
nginx_user: www-data

# A list of additional modules to install
nginx_extra_modules: []

# The path to the website root
nginx_web_root: /var/www/html

Notice the style: the variables are named clearly, and there are comments explaining what they do. This is a "self-documenting" role. A customer or teammate can look at this file and immediately understand how to use your automation.

vars/main.yml: The Internal Constants

If defaults/ is the lowest precedence, vars/ is near the top. Variables defined in role vars (Level 15) are very difficult to override. They will "win" against variables defined in the inventory or even in the vars: section of a playbook.

This high precedence is a protective measure. You use vars/main.yml for data that is environment-independent but logic-dependent.

When to Use vars/

Common candidates for vars/ include:

Fixed Paths: If your role always places a specific internal lock file at /var/run/my_role.lock, and changing that path would break your cleanup tasks, put it in vars/.

Naming Conventions: If your company mandates that all backup files must end in .bak.v1, define that extension in vars/.

OS-Specific Mapping: This is the most powerful use case for vars/. Different operating systems often use different names for the same package (e.g., httpd on Red Hat vs. apache2 on Ubuntu).

Example: OS-Specific Constants

Instead of a single main.yml, architects often use vars/ to handle cross-platform compatibility. You might have a task that includes a specific variable file based on the OS:

# tasks/main.yml
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: Ensure web server is installed
  package:
    name: "{{ apache_package_name }}"
    state: present

In this scenario, your vars/RedHat.yml would look like this:

---
apache_package_name: httpd
apache_service_name: httpd

And your vars/Debian.yml would look like this:

---
apache_package_name: apache2
apache_service_name: apache2

The user doesn't need to know what the package is called; they just want "Apache" installed. By putting these in vars/, you ensure the user doesn't accidentally try to "override" the package name to something that doesn't exist on that OS.

The Pecking Order: A Simplified Precedence

Ansible has 22 levels of variable precedence, which can feel overwhelming. As an architect, you don't need to memorize the entire list every day, but you must understand the "Big Four" that impact role development.

Think of it as a ladder. Variables higher on the ladder will step on (override) variables lower on the ladder.

Extra Vars (-e at command line): The Heavyweight Champion. Nothing beats this. (Level 22)

Role Vars (vars/main.yml): High precedence. Intended to protect internal role logic. (Level 15)

Play Vars (vars: in a playbook): Mid-range. This is where a user usually overrides settings for a specific play. (Level 12)

Role Defaults (defaults/main.yml): The baseline. Designed to be overridden. (Level 2)

The "Predict and Reveal" Challenge

Imagine you are a user running a playbook. You see a role that manages a database.

The role has db_port: 3306 in defaults/main.yml.

The role has db_service_name: mysql-server in vars/main.yml.

You write a playbook and try to override both:

---
- hosts: db_servers
  vars:
    db_port: 9000
    db_service_name: custom-db-service
  roles:
    - database_role

What happens?

db_port will be 9000. Because Play Vars (Level 12) are higher than Role Defaults (Level 2), your override "wins."

db_service_name will remain mysql-server. Because Role Vars (Level 15) are higher than Play Vars (Level 12), the role "protects" its internal constant and ignores your attempt to change it in the playbook.

This is exactly why customers get frustrated when everything is put in vars/. They try to change a setting in their playbook, but nothing happens. They feel like the role is "locked" or "broken."

The Advisory Lens: Avoiding the "Locked Role" Anti-Pattern

In your role as a technical advisor, you will frequently encounter customer codebases where every single variable is crammed into vars/main.yml. 

The Symptom

The customer complains that their roles aren't reusable. They say, "We have one role for the Dev Nginx server and a completely different role for the Prod Nginx server." 

The Diagnosis

When you look under the hood, you see that the Prod role has nginx_port: 443 hardcoded in vars/main.yml, and the Dev role has nginx_port: 80 hardcoded in its own vars/main.yml. 

Because they used vars/, they couldn't simply use one role and override the port in their inventory. They effectively "locked" the role to a specific configuration, forcing them to duplicate the entire role for every slight variation. This is a maintenance nightmare. If they find a bug in the Nginx configuration, they now have to fix it in two (or ten) different places.

The Architect’s Advice

When you see this, your recommendation should be:

Identify the "Environmental Variables": What changes between Dev, Staging, and Prod? (Ports, URLs, usernames, memory limits).

Move them to defaults/main.yml: Unlock the role.

Use Inventory or Playbook vars for overrides: Let the caller decide the values for those specific environments.

Reserve vars/main.yml for "Static Truths": Only keep things there that stay the same regardless of whether you are in Dev or Prod (like the fact that on RHEL, the Nginx config file is always located at /etc/nginx/nginx.conf).

Case Study: Designing a Reusable App Role

Let's look at how an expert architect structures a role for a custom application.

The "Flexible" approach (The Goal)

defaults/main.yml (The knobs for the customer):

---
app_version: "1.0.0"          # User might want a specific version
app_install_dir: "/opt/app"    # User might have a different disk mount
app_enable_debug: false        # User might need to troubleshoot

vars/main.yml (The internal logic):

---
# The internal name of the service as defined in the systemd unit file
# If the user changes this, the 'service' tasks in the role will fail.
internal_systemd_service_name: "enterprise-app-daemon"

# The URL of the internal artifactory — this is a company constant
app_source_repo: "https://artifactory.internal.com/repo/apps"

The Advisory Conversation

When explaining this to a customer, you might say:

"By putting the app_install_dir in defaults, we're giving your app teams the freedom to install this wherever their specific project requires. However, we've kept the internal_systemd_service_name in vars. We do this because the role's logic—the handlers that restart the app and the tasks that check its status—is hard-coded to look for that specific service name. If a user were to change that name, the role would effectively 'break' itself. We're protecting the user from making a change that requires a deep understanding of the role's inner workings."

Self-Documenting through Defaults

One final tip for the aspiring architect: your defaults/main.yml is your most important documentation. Most users of your role will never look at tasks/main.yml. They will open defaults/main.yml to see what they can change.

Bad Practice:

---
port: 80
user: nginx
dir: /var/www

(This is confusing. What port? What user? What if this role is used alongside another role that also has a variable named port?)

Architect Practice (Namespacing and Commenting):

---
# The port Nginx listens on. Default is 80.
# Change to 443 if using SSL in the 'nginx_ssl' role.
nginx_http_port: 80

# The OS user that owns the web content files.
nginx_web_user: www-data

Always prefix your variables with the name of the role (e.g., nginx_). This prevents "variable collision," where two different roles use the same variable name and cause unpredictable behavior.

Summary of the Contract

Feature

defaults/main.yml

vars/main.yml

Precedence

Level 2 (Lowest)

Level 15 (High)

Purpose

Public API / Overridable settings

Internal Constants / Protection

Who changes it?

The Role User (via Playbook/Inventory)

The Role Author (via Role code)

Example

web_port: 80

service_name: httpd

Advisory Note

Makes roles reusable across environments.

Prevents users from breaking core logic.

Mastering the Flow

You now understand how to organize your variables and how to set up the "contract" between your role and its users. By using defaults/ for flexibility and vars/ for stability, you create roles that are easy to use and hard to break.

Now that we have our data structure and variables in place, we need to think about how our role reacts to changes. If a variable changes a configuration file, how do we ensure the service restarts to pick up those changes? We don't want to restart the service every time the playbook runs—only when it's necessary. This leads us perfectly into our next topic: Handlers. In the next section, we’ll explore how to use notify, listen, and flush_handlers to manage these side effects with surgical precision.