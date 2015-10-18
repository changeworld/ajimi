# Ajimi

Ajimi is a tool to compare servers by their files to shows difference in their configurations.
It is useful to achieve some kind of a regression test against a server by finding unexpected changes to its configuration file after running Chef, Ansible, or your own provisioning tool.

'Ajimi' means 'tasting' in Japanese. It was developed for originally replacing the existing server with the Chef's cookbook, but can be used for a general purpose of comparing two servers.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'ajimi'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install ajimi

## Configuration

Generate a sample configuration file to current directory:

    $ ajimi-init

And then edit Ajimifile:

    $ (Your favorite editor) ./Ajimifile

A sample configuration looks like the following:

```ruby
# Ajimi configuration file

# source setting
source "source.example.com", {
  ssh_options: {
    host: "192.168.0.1",
    user: "ec2-user",
    key: "~/.ssh/id_rsa"
  },
  enable_nice: true
}

# target setting
target "target.example.com", {
  ssh_options: {
    user: "ec2-user",
    key: "~/.ssh/id_rsa"
  },
  enable_nice: false
}

# check setting
check_root_path "/"

pruned_paths [
  "/dev",
  "/proc",
]

ignored_paths [
  *@config[:pruned_paths],
  %r|^/lost\+found/?.*|,
  %r|^/media/?.*|,
  %r|^/mnt/?.*|,
  %r|^/run/?.*|,
  %r|^/sys/?.*|,
  %r|^/tmp/?.*|
]

ignored_contents ({
  "/root/.ssh/authorized_keys" => /Please login as the user \\"ec2-user\\" rather than the user \\"root\\"/
})

pending_paths [
  "/etc/sudoers"
]

pending_contents ({
  "/etc/hosts" => /127\.0\.0\.1/
})

```

The following arguments are supported in the Ajimifile:

* `source` - String (Required), Hash (Required): Source server's name and options. Options are as follows.
  * `ssh_options` - Hash (Required): SSH connection options
      * `host` - String (Optional): SSH hostname, FQDN or IP address. Default is name of `source`.
      * `user` - String (Required): SSH username.
      * `key` - String (Required): Path to user's SSH secret key.
  * `enable_nice` - Boolean (Optional): If true, the find process is wrapped by nice and ionice to lower load. Default is false.
* `target` - String (Required): Target server's name and options. Options are the same as source.
* `check_root_path` - String (Required): Root path to check. If "/", Ajimi checks in the whole filesystem.
* `pruned_paths` - Array\[String|Regexp\] (Optional): List of the path which should be excluded in the find process. Note that `pruned_paths` is better performance than `ignored_paths`/`pending_paths`.
* `ignored_paths` - Array\[String|Regexp\] (Optional): List of the path which should be ignored as known difference.
* `ignored_contents` - Hash{String => String|Regexp} (Optional): Hash of the path => pattern which should be ignored as known difference for each of the content.
* `pending_paths`- Array\[String|Regexp\] (Optional): List of the path which should be resolved later but ignored temporarily as known difference.
* `pending_contents` - Hash{String => String|Regexp} (Optional): Hash of the path => pattern which should be resolved later but ignored temporarily as known difference for each of the content.

## Usage

Ajimi is a single command-line application: `ajimi`.
It takes a subcommand such as `dir` or `exec` (Default subcommnad is `check`).
To view a list of the available commands , just run `ajimi help` with no arguments:

```bash
$ ajimi
Commands:
  ajimi [check]                         # (Default subcommand) Show differences between the source and the target server
  ajimi dir <path>                    # Show differences between the source and the target server in the specified directory
  ajimi exec source|target <command>  # Execute an arbitrary command on the source or the target server
  ajimi file <path>                   # Show differences between the source and the target server in the specified file
  ajimi help [COMMAND]                # Describe available commands or one specific command

Options:
  [--ajimifile=AJIMIFILE]      # Ajimifile path
                               # Default: ./Ajimifile
  [--verbose], [--no-verbose]
                               # Default: true
```

After setting your Ajimifle, Run the following command in order to verify the SSH connection:

    $ ajimi exec source hostname
    $ ajimi exec target hostname

And then, run ajimi command with `--find-max-depth` option:

    $ ajimi --find-max-depth=3 > ajimi.log

Check the diffs report in ajimi.log, and add roughly unnecessary paths to `pruned_paths` in Ajimifile.

Next, gradually increasing `find-max-depth=4, 5, ...`,

    $ ajimi --find-max-depth=4 > ajimi.log

Add known differences to `ignored_paths` or `pending_paths`.

After checking the difference of paths, then check the contents of files where the difference has been reported:

    $ ajimi --enable-check-contents > ajimi.log

Add known differences to `ignored_contents` or `pending_contents`,
and repeat until the number of lines of diffs report becomes human-readable.

Finally, resolve issues and remove `pending_paths` or `pending_contents`.

## Command reference

```bash
$ ajimi
Commands:
  ajimi [check]                         # (Default subcommand) Show differences between the source and the target server
  ajimi dir <path>                    # Show differences between the source and the target server in the specified directory
  ajimi exec source|target <command>  # Execute an arbitrary command on the source or the target server
  ajimi file <path>                   # Show differences between the source and the target server in the specified file
  ajimi help [COMMAND]                # Describe available commands or one specific command

Options:
  [--ajimifile=AJIMIFILE]      # Ajimifile path
                               # Default: ./Ajimifile
  [--verbose], [--no-verbose]
                               # Default: true
```

```bash
$ ajimi help check
Usage:
  ajimi [check]

Options:
  [--check-root-path=CHECK_ROOT_PATH]
  [--find-max-depth=N]
  [--enable-check-contents], [--no-enable-check-contents]
  [--limit-check-contents=N]
                                                           # Default: 0
  [--ajimifile=AJIMIFILE]                                  # Ajimifile path
                                                           # Default: ./Ajimifile
  [--verbose], [--no-verbose]
                                                           # Default: true

(Default subcommand) Show differences between the source and the target server
```

```bash
$ ajimi help dir
Usage:
  ajimi dir <path>

Options:
  [--find-max-depth=N]
                                       # Default: 1
  [--ignored-pattern=IGNORED_PATTERN]
  [--ajimifile=AJIMIFILE]              # Ajimifile path
                                       # Default: ./Ajimifile
  [--verbose], [--no-verbose]
                                       # Default: true

Show differences between the source and the target server in the specified directory
```

```bash
$ ajimi help file
Usage:
  ajimi file <path>

Options:
  [--ignored-pattern=IGNORED_PATTERN]
  [--ajimifile=AJIMIFILE]              # Ajimifile path
                                       # Default: ./Ajimifile
  [--verbose], [--no-verbose]
                                       # Default: true

Show differences between the source and the target server in the specified file
```

```bash
$ ajimi help exec
Usage:
  ajimi exec source|target <command>

Options:
  [--ajimifile=AJIMIFILE]      # Ajimifile path
                               # Default: ./Ajimifile
  [--verbose], [--no-verbose]
                               # Default: true

Execute an arbitrary command on the source or the target server
```

## Development and Test

    $ bundle install
    $ bundle exec ajimi-init
    (Implement some feature)
    $ bundle exec rake spec

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request


## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
