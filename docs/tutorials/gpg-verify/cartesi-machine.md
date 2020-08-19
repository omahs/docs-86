---
title: GPG Verify machine
---

:::note Section Goal
- build a machine that executes GPG signature verification from arbitrary document and signature data
- understand limitations of `ext2` file-systems for submitting dynamic data to a Cartesi Machine
:::


## Final execution script

Shifting our focus from understanding and testing GPG usage to an actual implementation of our DApp's Cartesi Machine, we must first of all acknowledge that our final machine cannot simply read the document and signature data from [pre-defined files](../ext2-gpg/#test-data). Rather, those two pieces of data should be read as separate input drives, so that they can be submitted in a smart contract request.

Unfortunately, as noted in the [Cartesi Machine section](../../../machine/host/cmdline#flash-drives), there is no direct way of generating `ext2` file-systems in a reproducible way, so these input drives have to be *raw*. This means that, instead of being mounted as file-systems, the drives' contents are to be read and written directly as plain bytes. Therefore, since we do not know in advance the exact size of the document or the signature, we will choose to *encode* the document and signature content lengths in the first two bytes of the input drives' data, so that we can correctly read the binary contents.

Aside from that, when writing the previous section's [Cartesi Machine example](../ext2-gpg/#cartesi-machine-with-gpg), you may have noticed that specifying all those command instructions as a single line was quite cumbersome. Thus, to tackle that issue and better organize our DApp, we will specify our final machine's commands in a separate *shell script* file, and include that file in the `ext2` file-system that our machine is already using to have access to the public key.

Back in the `gpg-verify/cartesi-machine` directory, create a file called `gpg-verify.sh` and make it executable:

```bash
touch gpg-verify.sh
chmod +x gpg-verify.sh
```

Now, edit that file and insert the following contents:

```bash
#!/bin/sh

# reads document data as a binary string whose length is encoded in the first 2 bytes, and stores it in file 'document'
dd status=none if=$(flashdrive document) | lua -e 'io.write((string.unpack(">s2",  io.read("a"))))' > document

# reads signature data as a binary string whose length is encoded in the first 2 bytes, and stores it in file 'signature'
dd status=none if=$(flashdrive signature) | lua -e 'io.write((string.unpack(">s2",  io.read("a"))))' > signature

# imports public key informing that it can be trusted (0xA86D9CB964EB527E is the key's LONG id)
gpg --trusted-key 0xA86D9CB964EB527E --import /mnt/dapp-data/descartes-pub.key

# verifies document signature
gpg --verify signature document

# writes gpg verify's exit status to output: 0 is success, 1 is failure, other values indicate error
echo $? > $(flashdrive output)
```

The first lines of the above script read the document and signature data from their respective input flash drives, and use a tiny Lua script to read the data considering the two initial bytes as the content length. The contents are then written to local files respectively called `document` and `signature`. After that, the script uses `gpg` to import the public key provided in the mounted `dapp-data` file-system, and finally verifies if the signature is valid for the given document. The final exit status is then written to the output drive.

## Final `ext2` file-system

With the execution script ready, we can now build the actual `ext2` file-system that we are going to use in our machine. This will be very similar to the process we did in the [previous section](../ext2-gpg/#building-an-ext2-file-system) for our tests, but now we will include the new `gpg-verify.sh` script we just wrote, while leaving out all of the test data.

First, create a directory called `ext2` within the current `gpg-verify/cartesi-machine` directory, and copy the public key and execution script files into it:

```bash
mkdir ext2
cp descartes-pub.key gpg-verify.sh ext2/
```

Now, use the `genext2fs` tool within `cartesi/playground` Docker image to build the `ext2` file:

```bash
docker run \
  -e USER=$(id -u -n) \
  -e GROUP=$(id -g -n) \
  -e UID=$(id -u) \
  -e GID=$(id -g) \
  -v `pwd`:/home/$(id -u -n) \
  -w /home/$(id -u -n) \
  --rm cartesi/playground:0.1.1 \
    genext2fs -b 1024 -d ext2 dapp-data.ext2
```

After the command completes. a file called `dapp-data.ext2` with the expected contents will be present in the current directory.


## Full machine implementation

After building our `ext2` file-system, we can then proceed to the final implementation of our Cartesi Machine. As done for all the [other tutorials](../../helloworld/cartesi-machine#cartesi-machine-for-the-hello-world-dapp), we will code a bash script to make it easy for us to build (and rebuild, if necessary) the final machine's *template specification*, and store it the appropriate location.

In order to that, create a `build-cartesi-machine.sh` file in the `cartesi-machine` directory:

```bash
touch build-cartesi-machine.sh
chmod +x build-cartesi-machine.sh
```

Now, place the following contents into it:

```bash
#!/bin/bash

# general definitions
MACHINES_DIR=.
MACHINE_TEMP_DIR=__temp_machine
CARTESI_PLAYGROUND_DOCKER=cartesi/playground:0.1.1

# set machines directory to specified path if provided
if [ $1 ]; then
  MACHINES_DIR=$1
fi

# removes machine temp store directory if it exists
if [ -d "$MACHINE_TEMP_DIR" ]; then
  rm -r $MACHINE_TEMP_DIR
fi

# builds machine (running with 0 cycles)
# - initial (template) hash is printed on screen
# - machine is stored in temporary directory
docker run \
  -e USER=$(id -u -n) \
  -e GROUP=$(id -g -n) \
  -e UID=$(id -u) \
  -e GID=$(id -g) \
  -v `pwd`:/home/$(id -u -n) \
  -w /home/$(id -u -n) \
  --rm $CARTESI_PLAYGROUND_DOCKER cartesi-machine \
    --max-mcycle=0 \
    --initial-hash \
    --store="$MACHINE_TEMP_DIR" \
    --flash-drive="label:dapp-data,filename:dapp-data.ext2" \
    --flash-drive="label:document,length:1<<12" \
    --flash-drive="label:signature,length:1<<12" \
    --flash-drive="label:output,length:1<<12" \
    -- $'date -s \'2100-01-01\' && /mnt/dapp-data/gpg-verify.sh'

# moves stored machine to a folder within $MACHINES_DIR named after the machine's hash
mv $MACHINE_TEMP_DIR $MACHINES_DIR/$(docker run \
  -e USER=$(id -u -n) \
  -e GROUP=$(id -g -n) \
  -e UID=$(id -u) \
  -e GID=$(id -g) \
  -v `pwd`:/home/$(id -u -n) \
  -h playground \
  -w /home/$(id -u -n) \
  --rm $CARTESI_PLAYGROUND_DOCKER cartesi-machine-stored-hash $MACHINE_TEMP_DIR/)
```

Comparing this to the [test Cartesi Machine](../ext2-gpg/#cartesi-machine-with-gpg) of the previous section, we can see that we now have individual input flash drives for the document and signature data, as well as an output flash drive. Futhermore, the command line executed is a lot simpler, since we now moved most of the complexity into the `gpg-verify.sh` script.

At this point, the machine template can be built and appropriately stored in the [Descartes SDK environment](../../descartes-env) by typing:

```bash
./build-cartesi-machine.sh ../../descartes-env/machines
```

Giving an output such as this:

```
0: 368aca58c884ef4ce2fa5c58b3f859ce8d001e725706f2d1e329fa1d7ae72f52

Cycles: 0
Storing machine: please wait
```

It is important to note here that, contrary to our other tutorials, the resulting template hash produced will be different from the one presented above, even though the machine specification and input contents are apparently identical. This happens because the hash captures the *complete initial state* of the machine and, as noted in the beginning of this section and explained in the [Cartesi Machine host perspective](../../../machine/host/cmdline#flash-drives), using the `genext2fs` tool to build a new `ext2` file with the *same contents* will actually always lead to a slightly *different* file.

As such, to get the exact same result you will need to download the very same `ext2` file that was used to build the machine when this tutorial was written, which is actually [available in the Descartes Tutorials GitHub repo](https://github.com/cartesi/descartes-tutorials/tree/master/gpg-verify/cartesi-machine). Thus, we can finish off this section by executing the following commands to retrieve that file and then rebuild the machine template using it:

```bash
wget https://github.com/cartesi/descartes-tutorials/tree/master/gpg-verify/cartesi-machine/dapp-data.ext2
./build-cartesi-machine.sh ../../descartes-env/machines
```

Which should now yield the exact same output as above.