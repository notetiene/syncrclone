# syncrclone

Robust, Configurable, Bi-Directional sync between *any* two rclone remotes with automatic conflict resolution and backups.

syncrclone is in beta. It has been tested with a variety of backends but by no means all of them. And only has been real-world tested with a few. See [testing notes](docs/tests.md) for some details.

## Features

* Configurable rename tracking even for remotes with incompatible hashes 
* Configurable file comparison and conflict resolution. 
    * Does **not** require ModTimes. Can use hashes or size for comparison (and move tracking)
* Entirely non-interactive
* All opperations that modify/delete files include a backup by default
* Robust to interruption
* Optional Locking system
* Dry-Run mode
* Extensive tests
* Directly uses rclone's powerful [filtering](https://rclone.org/filtering/)
* Efficient when possible. Optionally avoids a second file-listing after synchronization. 

## Installation and Usage

First, [install rclone](https://rclone.org/install/). Then, you must have **python 3.6+** installed. There are many options. I am a fan of [miniconda](https://docs.conda.io/en/latest/miniconda.html).

Install syncrclone:

    $ python -m pip install git+https://github.com/Jwink3101/syncrclone
    
Configure rclone: I prefer to specify a config file using `--config rclone.cfg`. Add the remotes you wish to sync

Initiate syncrclone: (see "Local and Remote Mode" below)

    syncrclone --new config.py

Modify the config code. It is fully documented but also see [config tips](docs/config_tips.md). If you use your own rclone config file above, make sure to include 

    rclone_env = {'RCLONE_CONFIG': 'rclone.cfg'}

or
    
    rclone_flags = ['--config','rclone.cfg']

It is a good idea to read the *entire* config file and set as needed.

Now run it! You do not need to do anything special even though it is the first run. It's just that all files will be considered new.

    $ syncrclone config.py

That's it!

**WARNING**: The config file is directly executed and is assumed trusted. If you keep config files in sync, be careful for malicious code

### Local and Remote Mode

syncrclone offers a *convenience* mode for local repos. It is functionally identical but makes calling and set up easier. The differences are:

* Local mode will look for `.syncrclone/config.py` (searching upwards) while remote mode expects the path of the sync config to be specified.
* When creating a `--new` sync config, it will put it in `.syncrclone/config.py`. And the `remoteA` will be populated as `../`

To work in local mode, specify a directory in the command line and it will search upwards for `.syncrlone/config.py`. To work in remote mode, specify the path to the config script. Recall that rclone is always calls from the directory of the sync config therefore, when in local mode, it is being called from `.syncrclone`. Specify other paths as needed.

For example
```bash
$ cd /path/to/local/files
$ syncrclone
```
is the same as the following:
```
$ cd /path/to/local/files
$ syncrclone .syncrclone/config
```
or even deeper:
```bash
$ cd /path/to/local/files/deeper/sub/dirs/
```
Then you can do either of the following
```bash
$ syncrclone   # Will automatically find it
$ syncrclone ../../../.syncrclone/config.py 
```

### rclone Versions

This tool uses some newer rclone flags so it is always good to make sure you're using the newest rclone. There is a small chance that an rclone change could break something so you *may* have to use a previous version until it is updated.

## Filtering, etc

All filtering is handled by rclone's filtering. See their [detailed documentation](https://rclone.org/filtering/).

Filter flags should be set *only* in the config `filter_flags` section. There are many options for filtering such as `--exclude path`. See note below about `--exclude-if-present`.

Remember that rclone is called from the same directory as the config file so make sure paths for flags such as `--filter-from` are correctly specified.

See more on Filters in the [config tips](docs/config_tips.md)

### Exclude if Present Filters: A Warning

Filters work well since they are applied to both sides. This means that changing a filter will make the files look like it was deleted on *both* sides and nothing will happen.

However, the [`--exclude-if-present`][eifp] filter is **very dangerous** if the excluded file is added on one side only *after* files in the directory are in sync. It will cause rclone to skip the directory on one side and appear like it was deleted. 

There *are* safe ways to use `--exclude-if-present`. For example, you can place exclude file in place on *both* sides before syncing. (Or placing it without adding the filter, syncing, and then adding the filter). Or, if the files have *never* been synced before (i.e. it's a new directory with the exclude file) then nothing bad will happen.

Note that if `--exclude-if-present` is found in the `filter_flags`, a warning will be emitted.

[eifp]:https://rclone.org/filtering/#exclude-directory-based-on-a-file

## Non-ModTime comparisons and conflicts

The default is to decide if files need to sync by comparing ModTime (or `mtime`). However, you can also compare by size or hash (robust).

If you compare by size or hash, you can still resolve conflicts with modification time. Do note that not all remotes support ModTime and/or it may not be reliable. If using one of those types of remotes, do not use `newer`, `older`, or `newer_tag` conflict resolution. See [remote overview](https://rclone.org/overview/) for details.

If your remote doesn't store hashes and must recalculate them (e.g. local, sftp), use `reuse_hashes(A/B)` if desired to only recalculate as needed. See the config file

## Differences from PyFiSync

[PyFiSync](https://github.com/Jwink3101/PyFiSync) was originally designed to use ssh+rsync for the remote. rsync is able to efficiently transfer small changes to large files so *a lot* (!!!) of effort went into tracking moves even on changed files. And, I know I would always have `mtime` and `inodes` (No Windows support) and when in macOS, `birthtime`.

I later added rclone remotes. rclone is an amazing piece of software but has no way to sync deltas (which is reasonable given cloud infrastructure) so all of the work to track modified files was wasted! And not even possible on the remote side.

syncrclone was designed exclusively for rclone which means that I didn't have to try to track moves with modifications. As such, the algorithm is a *lot* simpler and a lot of edge cases are eliminated. The key difference with this algorithm is that files that match are removed from consideration right away. Then moves are only tracked via files that are (a) new, (b) match a previous file, and (c) are marked for deletion on the other side. Furthermore, while risky, syncrclone *can* compare sides by file size alone (it can also track moves by file size alone but that is really risky!) so all rclone remotes can now be used.

## Additional Documents

Some additional docs:

* [Configuration tips](docs/config_tips.md), including setting rclone flags
* [The algorithm](docs/algorithm.md)
* [Miscellaneous details](docs/misc.md)
* [Testing](docs/tests.md). Also includes how to test with other remotes.
* [Changelog](docs/changelog.md)

## Alternatives

File sync is surprisingly opinionated in many ways including use of config files vs pure CLI, how to handle conflicts, how to handle moves, how/if to backup files, etc.

I wrote syncrclone partially because I wanted a sync tool that works *exactly* the way I prefer! Same reason I wrote [PyFiSync](https://github.com/Jwink3101/PyFiSync). But there are alternatives out there. To name a few:

* [PyFiSync](https://github.com/Jwink3101/PyFiSync) -- This is my own tool and I talk about it above. If you can run it on both machines and use rsync, it is a great option
* [rclonesync-V2](https://github.com/cjnaz/rclonesync-V2) -- Different philosophy but also pretty powerful and well thought out! Pure CLI without config files.
* [rsinc](https://github.com/ConorWilliams/rsinc) -- Another rclone-to-rclone sync tool. Also seems well designed and thought out. Uses a config file. Also implements its own exclusion engine.
* [FreeFileSync](freefilesync.org) -- GUI program that doesn't use rclone. Good choice for those only looking to use local or SFTP and do not wish to deal with CLI.

I may have some details wrong. Please let me know and I will fix them.
