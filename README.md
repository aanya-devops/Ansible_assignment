# SonarQube Ansible Role - Assignment 5

## Project Title

**sonarqube - OS-Independent, Version-Pinned Ansible Role for SonarQube Installation**

## Project Description

This project implements a single Ansible role, `sonarqube`, that installs and configures SonarQube across multiple Linux distributions from one codebase. The role targets three nodes of different OS families in the same run - `debian1` (Debian), `ubuntu1` (Ubuntu), and `redhat1` (RedHat) - and installs a specific, pinned version of SonarQube along with its Java prerequisite, using OS-specific task files that are pulled in conditionally based on facts gathered from each host.

## Objective

The main objectives of this assignment are:

- Build a role that installs a specific, pinned version of the assigned tool (SonarQube) rather than "whatever is latest."
- Make the role OS independent, so it works correctly on both Debian-family and RedHat-family systems.
- Externalize all configuration into variables instead of hardcoding values into tasks.
- Use Jinja2 templates for the tool's configuration files, with all dynamic values driven by variables.
- Include handlers for restart/reload actions, kept separate from the main task file rather than mixed into it.
- Let the user choose to run the role against CentOS/RedHat, Ubuntu/Debian, or both together in the same play.

## Features

- **Version-pinned installation** - the SonarQube package is downloaded directly from a specific versioned URL (`sonarqube-10.6.0.92116.zip`) rather than tracking a "latest" pointer, so the installed version is predictable and repeatable.
- **OS independence via fact-based task inclusion** - the role gathers `ansible_distribution` for every host first, then includes either `tasks/debian.yml` or `tasks/redhat.yml` for the prerequisite installation step, so the same role adapts its package-manager logic (`apt` vs `yum`/`dnf`) per host automatically.
- **Variablized configuration** - settings such as install paths, service limits, and OS-specific package lists are defined as role variables rather than being hardcoded in the tasks.
- **Jinja2 templates** - SonarQube's configuration file and systemd service unit are generated from templates, with variables substituted in for values like installation directories, memory/file-descriptor limits, and paths.
- **Handlers kept separate from tasks** - service reload and restart actions live in the role's `handlers/main.yml`, triggered by `notify` from the relevant configuration tasks, rather than being written as inline tasks.
- **Directory management** - dedicated directories (`/opt/sonarqube/logs`, `/opt/sonarqube/temp`) are created with the correct ownership as part of the role, independent of the extracted package contents.
- **Idempotency demonstrated via re-run** - the role was intentionally re-run after backing up the role directory, showing tasks correctly reporting `ok` (already satisfied, like the Java 17 download being skipped since it was already present) instead of repeating work unnecessarily.
- **Runs against one OS, multiple OS, or all together** - since the role is included from a single play targeting the `all` group, the user can limit the run to just `redhat1`, just the Debian-family hosts, or all three nodes together, simply by adjusting the inventory group or `--limit` used with `ansible-playbook`.

## Tech Stack

- Ansible (role-based, run from a Python virtual environment)
- Debian, Ubuntu, and RedHat target nodes
- SonarQube 10.6.0 (Community)
- OpenJDK 17 (SonarQube's Java prerequisite)
- Jinja2 templating
- systemd (for the SonarQube service unit)

## Installation Steps

1. Activate the Ansible virtual environment and move into the assignment directory:
   ```bash
   cd Ansible_35/Assignment5
   ```

2. Confirm the `inventory` file lists all three target hosts (`debian1`, `ubuntu1`, `redhat1`).

3. Verify connectivity and check each host's distribution:
   ```bash
   ansible all -i inventory -m ping
   ansible all -i inventory -m setup -a 'filter=ansible_distribution'
   ```

4. Validate the playbook syntax before running it:
   ```bash
   ansible-playbook -i inventory site.yml --syntax-check
   ```

5. Run the playbook against all hosts, a single OS family, or any subset using `--limit`:
   ```bash
   ansible-playbook -i inventory site.yml
   ansible-playbook -i inventory site.yml --limit redhat1
   ansible-playbook -i inventory site.yml --limit "debian1,ubuntu1"
   ```

## Folder Structure

```text
Assignment5/
│
├── inventory
├── site.yml
├── roles/
│   └── sonarqube/
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── debian.yml
│       │   └── redhat.yml
│       ├── templates/
│       │   ├── sonar.properties.j2
│       │   └── sonarqube.service.j2
│       ├── handlers/
│       │   └── main.yml
│       ├── defaults/
│       │   └── main.yml
│       ├── vars/
│       │   ├── debian.yml
│       │   └── redhat.yml
│       └── meta/
│           └── main.yml
└── images/
    ├── 01-ping-and-os-facts.png
    ├── 02-whoami-syntax-check-playbook-start.png
    ├── 03-download-sonarqube-package.png
    ├── 04-configure-service-handlers-recap.png
    ├── 05-second-run-os-conditional-tasks.png
    ├── 06-sonarqube-browser-ubuntu.png
    └── 06-sonarqube-browser-redhat.png
```

## How to Run the Project

### Verify Connectivity and Detect OS per Host

```bash
ansible all -i inventory -m ping
ansible all -i inventory -m setup -a 'filter=ansible_distribution'
```

Confirms all three hosts (`debian1`, `ubuntu1`, `redhat1`) are reachable, and that Ansible correctly identifies each one's distribution - `Debian`, `Ubuntu`, and `RedHat` respectively. This distribution fact is what later drives which OS-specific task file gets included for each host.

<img width="1500" height="712" alt="01-ping-and-os-facts" src="https://github.com/user-attachments/assets/5b129cd3-a818-481e-a32e-6214a72a56b8" />


### Confirm Privilege Level, Syntax Check, and Start the Playbook

```bash
ansible all -i inventory -b -m shell -a 'whoami'
ansible-playbook -i inventory site.yml --syntax-check
cp -r roles/sonarqube roles/sonarqube_backup
ansible-playbook -i inventory site.yml
```

Confirms tasks run as `root` on every host, validates the playbook syntax, backs up the role directory before making any further changes, and begins the first full run. Facts are gathered, OS-specific variables are loaded per host, and the correct OS task file (`redhat.yml` for `redhat1`, `debian.yml` for `ubuntu1` and `debian1`) is included automatically based on each host's detected distribution.

<img width="1530" height="871" alt="02-whoami-syntax-check-playbook-start" src="https://github.com/user-attachments/assets/4eb145dd-1811-4454-8f60-6ccfab927878" />


### Download the SonarQube Package (Version-Pinned)

```bash
ansible debian1 -i inventory -b -m get_url -a "url=https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.6.0.92116.zip dest=/tmp/test-sonarqube.zip mode=0644"
ansible debian1 -i inventory -b -m shell -a "ls -lh /tmp/test-sonarqube.zip"
```

Confirms the role downloads an exact, versioned SonarQube release (`10.6.0.92116`) rather than a "latest" build, and that the ~698 MB archive is fetched successfully with the correct checksum.

<img width="1523" height="532" alt="03-download-sonarqube-package" src="https://github.com/user-attachments/assets/6c16677a-d008-4b2e-b4d9-ca7c52c52906" />


### Configure SonarQube, Service, and Trigger Handlers

Continuing the same run: SonarQube's configuration file and systemd service are generated from templates, resource limits are applied, and the service is enabled and started. Because `debian1`'s configuration actually changed, the `Reload systemd` and `Restart SonarQube` handlers are triggered at the end of the play - while `redhat1` and `ubuntu1`, whose configuration didn't change on this pass, correctly skip re-running those handlers. The play recap confirms all three hosts finished with zero failures.

<img width="1526" height="816" alt="04-configure-service-handlers-recap" src="https://github.com/user-attachments/assets/3007bb2d-d306-4ac8-98f0-73c55e34ca13" />


### Second Run - OS-Conditional Tasks and Idempotency

```bash
ansible-playbook -i inventory site.yml
```

A second run against all three hosts. Debian-family hosts go through `apt` cache update and Java installation via their own task file, while `redhat1` goes through its own prerequisite and package installation path. Notice the `Download Java 17` task now reports `skipping` for `redhat1`, since Java was already installed correctly in the previous run - confirming the role is idempotent.

<img width="1526" height="857" alt="05-second-run-os-conditional-tasks" src="https://github.com/user-attachments/assets/9029029e-4823-4baa-a8fa-b797a2ec9b77" />


### Verify SonarQube in the Browser

**Ubuntu node**

Accessing `ubuntu1`'s public IP on port 9000 confirms SonarQube is installed and reachable, landing on the "How do you want to create your project?" screen.

<img width="1170" height="595" alt="06-sonarqube-browser-ubuntu" src="https://github.com/user-attachments/assets/47c3bc17-26e1-43ee-8ad5-fa8a32476b54" />


**RedHat node**

The same verification repeated against `redhat1`'s public IP, confirming SonarQube came up correctly on the RedHat-family host as well, using the exact same role.

<img width="1758" height="895" alt="06" src="https://github.com/user-attachments/assets/c120a8a2-ed3f-42b9-a223-04527f8396b0" />


## Learning Outcome

Through this assignment, I learned:

- How to make a single Ansible role OS-independent by gathering `ansible_distribution` first and using `include_tasks` to pull in the correct OS-specific task file per host, rather than writing separate roles per OS.
- How to pin an exact software version by downloading a fully versioned URL instead of relying on a package repository's "latest" release, making installations reproducible.
- How to keep a role's configuration flexible by pushing values like paths, limits, and package lists into `defaults/` and OS-specific `vars/` files instead of hardcoding them into tasks.
- How to use Jinja2 templates to generate both an application configuration file and a systemd service unit, with all environment-specific values substituted in from variables.
- Why handlers belong in their own `handlers/main.yml` and are triggered via `notify`, rather than being written as regular tasks - so a restart only happens when something actually changed.
- How `--syntax-check` and a full run complement each other, and why backing up a role directory before a second run is a useful safety habit when iterating.
- How to confirm idempotency directly from the playbook output - a task reporting `skipping` or `ok` instead of `changed` on a second run is proof the role isn't doing unnecessary work.
- How `--limit` can be used to run the exact same playbook against a single OS family or against every node together, satisfying the requirement to support CentOS, Ubuntu, or both without maintaining separate playbooks.

## Author

**Aanya Tyagi**
