I've recently switched over to using the
fantastic [autoenv](https://github.com/kennethreitz/autoenv) to automatically
activate my anaconda environments and set necessary environment variables when I
enter a directory on my terminal. You basically write some bash code in a `.env`
file, put it into a directory, and autoenv will automatically run `.env` when
you enter the directory or any of its subdirectories.

However, I found that putting just `source activate desired_environment` in my
`.env` (to activate the `desired_environment` conda environment) made my shell
_very_ slow --- I'd have to wait ~2 seconds after issuing a `cd` into a
directory with a `.env` file (or a subdirectory of one).

The following bash snippet makes activating conda environments with `autoenv` a
lot faster:

```
current_environment=""
environment_to_activate=duplicate

# $CONDA_PREFIX is non-empty when in an environment
if [[ $CONDA_PREFIX != "" ]]; then
  # Get the name of the environment from the path
  current_environment="${CONDA_PREFIX##*/}"
fi

if [[ $current_environment != $environment_to_activate ]]; then
  # We are not in the environment to activate, so activate it.
  source activate $environment_to_activate
fi
```

The snippet basically checks if you're already in the conda environment you want
to activate (called `test` in this case, and assigned to
`environment_to_activate`), and doesn't rerun the slow `activate` script if you
are. Handy!

To use this snippet, just drop it into your `.env` file and replace `test` with
the name of whatever environment you want to activate; your shell should feel a
lot less slow.
