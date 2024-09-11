# Users

## User permissions

By design, users will not have `sudo` access on the server (except to act as the
conda user). However, `sudo` or `root` access should not be required to get work
done.

Users will be added with a few resources available to them:

* Global conda environment that is shared amongst lab members
* Local conda environment for personal or testing work
* A home directory `/home/<username>`
* Write access to their own `scratch` directory (`/scratch/<username>`)
* Write access to their own `work` directory (`/work/<username>`)
* Write access to their own `home` directory (`/home/<username>`)

The `work` directory is intended for long term storage of data. This directory
is on disks that are slower (relatively) than other disks on the machine, but
offer a higher capacity. (~18T of storage for the whole lab).


Currently, there are no quotas or per-user limits on how much data can be stored
in their directories. This means that lab members will need to coordinate to
make sure they're not using too much space.

To check the amount of space you're using in your `/home`, `/scratch` and
`/work` directory, you can run the following command:

```
du -sh /home/<username> /scratch/<username> /work/<username>
```

NOTE: if you have many small files (especially on the work directory) this
command might take a while to run.

## Adding new users

To add new users, Dr. Weber can run a script that is contained inside of his
home directory called `new-lab-user`. The script is in this directory should it
need modification: `/home/weber/.local/bin/new-lab-user`. To view the usage of
the script, run `new-lab-user --help`. To add a new user with the defaults, run:

```
new-lab-user <username>
```

This will generate a new user with the permissions and directories described
above. A user's default password upon generation is `password`. Upon logging in
for the first time, users should run `passwd` to update to a better password.
