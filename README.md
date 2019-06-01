# EC2 NVMe Block Device Configuration

Tools and configuration for working with Amazon EC2 NVMe block device metadata and providing stable device names across boots. With the introduction of instance-store and EBS-backed NVMe devices, there is no guarantee of stable `nvme?n?` device names (eg `/dev/nvme0n1` on first boot might appear as `/dev/nvme2n1` on a subsequent boot if there are additional NVMe devices present).

Original work by Amazon, adapted and extended by Netflix.

Relevant EC2 documentation:

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/nvme-ebs-volumes.html

## Tools

### ec2nvme-id

`ec2nvme-id` is adapted from a tool of the same name provided by Amazon for use in Amazon Linux that examines NVMe metadata and extracts embedded `xvd*` or `sd*` device names as requested during attachment in order to provide stable device names aligning with the user requested block device. 

#### Changes from the Original

The script was ported to Python 3 and is known to work against Python 3.5 and later. No dependencies beyond the Python stdlib are required. 

The `ctypes` definitions were fleshed out to fully reflect the structures provided by the kernel and the overall source was cleaned up a bit to make it usable both as a CLI tool and as a library for other tools if necessary.

For EBS NVMe devices, the original tool only returns the exact embedded block device alias. For example, if one asked for an EBS volume at `/dev/xvdb` and another at `/dev/sdc`, querying the volumes would only return that explicit alias. Given that one can mix-and-match `xvd*` and `sd*` in requests, for consistency, the tool was modified to return both `xvd` and `sd` aliases. While the EC2 API will *accept* requests for EBS volumes at `/dev/xvdb` and `/dev/sdb`, only one or the other will actually succeed, so we can safely set up both aliases as we can never have two distinct block devices at `xvdN` and `sdN` for any `N`.

In the original tool, there was no special consideration given to instance-store NVMe devices as one might find on `i3.*` instance types or the `*5d` Nitro instance types. When one launches those instances, instance-store NVMe devices are always attached and ready regardless of any ephemeral block device mapping configuration attached to the AMI, instance, or launch configuration. While these devices do embed a unique serial number and manufacturer string identifying them as NVMe instance-store devices, there is no embedded block device alias. To help identify these disks, a synthetic friendly block device alias is established for each instance-store NVMe device in the form of `/dev/ec2_ephemeral_nvmeX` where `X` corresponds to `X` in `/dev/nvmeXn1`.

  **NOTE**: These friendly aliases are *not* currently guaranteed to be stable across reboots but do help one distinguish between EBS- and instance-store-NVMe devices at-a-glance. A future update will make these friendly aliases stable across reboots by using a sorted list of the embedded serial numbers in instance-store NVMe devices to establish a stable numeric order for the `ec2_ephemeral_nvmeX` friendly names. This relies on the fact that one cannot currently add or remove instance-store NVMe devices during an instance's lifetime and the stability guarantee no longer holds if that changes.

### ec2nvme-nsid

`ec2nvme-nsid` extracts the ns number from a given NVMe kernel device name for use in configuration of NVMe ns device links under `/dev/disk/by-id`.

#### Changes from the Original

This variant is functionally identical but leans on shell parameter expansion over subshells and pipes.

## Configuration

The udev rules file uses the above tools and establishes the NVMe device symlinks on instance boot. It differs from Amazon's version in that it configures both `sd` and `xvd` aliases and handles instance-store volumes as described above.
