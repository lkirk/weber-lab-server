# Conda

The conda configuration on this server is designed with two goals in mind:

1. Shared environments for common workflows
1. Ability to create conda environments in user home directories

To avoid any accidents with the common environments, users do not have direct
write access to the global conda environment. In addition, the `base`
environment is protected so that users cannot accidentally install packages that
will conflict with other environments or mess up the conda installation on the
server.

## Setting up conda for a user

For the most part, you'll likely want to install packages from `conda-forge` or
`bioconda`. The easiest way to make this a default is to run the following
commands:
```
conda config --prepend channels conda-forge
conda config --prepend channels bioconda
```

## Activating environments

To access all of the software installed into a conda environment, simply run:

```
conda activate <env name>
```

You should see the environment name prepended to your terminal prompt:
`(myenv) user@stickleback ~ %`

## Adding user environments

User environments will be installed in a user's home directory. They are private
to a user (see shared environments for creating shared environments). See the
conda documentation for [managing
environments](https://conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html)
for more information. All of the info on this page is applicable to user and
shared environments, you will just need to execute the commands as conda (see
below).

## User permissions

Users have the permissions to run commands as the `conda` user, which has
permissions to alter shared conda environments. NOTE: if you're not creating an
environment that will be shared by the lab, then you don't need to use `sudo`.

If you don't want to type `sudo -iu conda` every time, you can make a shell
alias (in your `.bashrc` or `.zshrc`):

```
alias as-conda='sudo -iu conda'
```

With this you can run;

```
as-conda conda <cmd>
```

If, for some reason, you need to become the conda user to perform a more complex
task, you can run `sudo su conda`.

## Creating new shared environments

Create a new environment by running the following command

```
sudo -iu conda conda create --name <env name>
```

Install packages into the shared conda environment. Note the `-n` flag, which
denotes the name of the environment you're installing into.
```
sudo -iu conda conda install -n <env name> <list of packages>
```

Once that's been created, you can activate the environment with:

```
conda activate <env name>
```

Note that you don't need sudo to activate a shared environment, only to modify
it (which you should do with care).


## Protecting an environment

If there's an environment that you want to "lock", meaning you want to protect
it from accidental modification, there is a plugin installed that allows for the
protection of environments. To protect an environment, run:

```bash
conda protect <env name>
```

If you try to install packages into the environment, you'll see that it is
locked. To unlock, run the above command again. More information about this
plugin can be found on the [project
page](https://github.com/conda-incubator/conda-protect).
