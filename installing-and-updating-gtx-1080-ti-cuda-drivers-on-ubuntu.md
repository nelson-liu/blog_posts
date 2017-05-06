I recently had to figure out how to set up a new machine with NVIDIA's new GTX
1080 Ti graphics card for use with CUDA-enabled machine learning libraries, e.g.
Tensorflow and PyTorch; since the card (as of this writing) is relatively new,
the process was pretty involved. The same tricks should also work for the newer
Titan Xp graphics card.

### 1. Install CUDA without the driver

I couldn't just install CUDA and have it work, since CUDA 8.0 comes with a
driver version (`375.26`) that doesn't support the GTX 1080 Ti. As a result,
installing CUDA from `apt-get` doesn't work since it installs this driver
version. Thus, **you have
to
[install with the runfile](http://docs.nvidia.com/cuda/cuda-installation-guide-linux/#runfile),
to opt-out of installing the driver.**

When running the installer, make sure to not install the driver that comes with
CUDA. We'll install the driver with `apt-get` in the next step.

**Post Install Notes** (Thanks to Jake Boggan for mentioning this in the
comments): After installing, check that the CUDA folders are where you expect
them to be (usually `/usr/local`). The CUDA installer creates a symlink at
`/usr/local/cuda` that automatically points to the version of CUDA installed.

Make sure to add `/usr/local/cuda/bin` to your `$PATH`, and
`/usr/local/cuda/lib64` to your `$LD_LIBRARY_PATH` if you're on a 64-bit machine
/ `/usr/local/cuda/lib` to your `$LD_LIBRARY_PATH` if you're on a 32-bit
machine. There's a bit more info at
the
[CUDA docs](http://docs.nvidia.com/cuda/cuda-installation-guide-linux/#post-installation-actions),
but the paths will likely differ based on version so be sure to manually verify
that the folders you're adding to the environment variables exist.
 
### 2. Installing the driver with apt-get

To install the driver with `apt-get`, I used the
Ubuntu
[graphics-drivers PPA](https://launchpad.net/~graphics-drivers/+archive/ubuntu/ppa).
This method isn't officially supported by NVIDIA, but it seems to work well for
many people.

At
the
[graphics-drivers PPA homepage](https://launchpad.net/~graphics-drivers/+archive/ubuntu/ppa),
there's a listing of the various graphics drivers that they offer; check
the [NVIDIA download website](http://www.nvidia.com/Download/index.aspx) to
figure out what version of the driver you need for your card. If it's in the
PPA, great! If not, you unfortunately have to wait for them to add it. They're
pretty timely, though.

Add the PPA to `apt-get` and update the index by running:
```bash
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt-get update
```

Now, we use it to install the desired driver versions (as of this writing):

For 1080 Ti: Major version `378`:
```bash
sudo apt-get install nvidia-378
```

For Titan Xp or 1080 Ti: Major version `381` (beta as of this writing)
```bash
sudo apt-get install nvidia-381
```

Reboot your computer, and the GPU should run on the new driver. To verify, run
`nvidia-smi` and confirm that the `Driver Version` at the top of the output is
what you expect and that the rest of the information looks good.

You should now be able to fire up Python
and
[test that it works with Tensorflow](https://www.tensorflow.org/tutorials/using_gpu#logging_device_placement) or
your favorite deep learning framework.

### For the future: updating the apt-get drivers

It's pretty easy to upgrade the drivers to a different version.

First, remove the old drivers:
```
sudo apt-get purge nvidia*
```

Now, just install the new driver with the PPA as detailed above and reboot.
