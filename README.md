# linux-custom-sysctls
Documentation for how to create custom sysctl variables in the linux kernel. There was a surprising lack of information when I was looking for how to do this myself, so I documented the process I followed.

As with any kernel documentation, the code it references is subject to change at any time. I will most likely _not_ have time to keep this up to date, so feel free to open a pull request, updating an old file or adding a new one.

Additionally, the case where I needed this functionality is pretty specific to the network stack in v4.19.68. It should be _relatively_ extensible, but kernel networking does like to do a lot of things differently than everything else. Once again, open a PR with a new file if the steps are different enough to justify it.

## Links
#### v4.19.68
* [Networking](sysctl-net-v4.19.68.md)
