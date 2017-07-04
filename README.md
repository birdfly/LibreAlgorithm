# LibreAlgorithm
Instructions on how to set a system that will allow open source projects to use LibreLink algorithm

# Introduction

Many users of libre have been suffering from the missing of alerts. Many open source projects have 
solved this issue but have failed to repeat the official abbott glucose numbers.

Other users who wanted to use LibreLink with sensors that they bought from abbott have found that their
sensors are not supported.

In this readme, I'll explain how to solve these two issues:

# Solution overview (details will follow)

1. Get a good old nexus 5.
2. Root it.
3. Replace the kernel. (This sounds scary, but this is the easiest step here, I'll suply
the kernel, you don't have to build it).
4. Install LibreLink on the phone.
5. On the first hour start your sensor with LibreLink.


# Every time that you want to know your bg, do the following:
1. Using your prefered software (for example xDrip), scan the sensor, get the scan
data in a file.
2. Attach GDB to the LibreLink (This probably means that you need the nexus 5 connected to a pc).
3. Put a breakpoint in the function isPatchSupported, make sure it returns true.
4. The new kernel has a modified NFC driver that will tell the system that you have just scanned a sensor.
Librelink will start working
5. press cont in the debugger. It will stop on the next breakpoint.
6. Load the files from step 1 into the program.
7. press cont in the debugger. Librelink will show your data on the screen.
8. The BG data will be stored in a sqlite file called sas.db Send it back to xDrip.


# How can this be made easier:

## Can we run without the debugger?
If we find a solution to the unsupported sensors, we can change the driver to insert
the scanned data directly, in that case the process will be much easier.

## Can this work without a computer attached? 
Theoretically yes, since gdb can run on the phone. Never tested it, so I don't know if it will work.

## I have another phone, can I use it?
Well, if you know how to build a kernel to your system that is possible. <br />
There are also other approaches to hook android calls. For example Xposed.
I don't know, but if they work, they are probably simpler than replacing the kernel. 
(They will still not solve the need to replace the kernel for unsupported sensors).

## Can one system support more than one sensor?
(In other words, can one build a server that will get scanned data and serve more than one person).
Yes, this will probably involve shutting down librelink, replacing the sas.db file
with data of that user.

## Will the phone let abbott know that I'm using this sw.
The nexus 5 can be without a sim and not connected to wifi, so abbott will not be
able to look at your personal data.  You can connect to it using adb.

#The details (will be completed):

1. Psadocode to scan the sensor:
```java   
        @Override
        protected void onPostExecute(Tag tag) {
            Log.d(TAG, "onPostExecute!!!");
            tag_discovered = false;
            Home.startHomeWithExtra(context, null, null);
            Home.staticBlockUI(context, false);
        }

        @Override
        protected Tag doInBackground(Tag... params) {
            Tag tag = params[0];
            //NfcOsHandle handle = this.osFunctions.getCommunicationHandle(tag);
            NfcV internalHandle = NfcV.get(tag);
            
            try {
                internalHandle.connect();
                Log.e(TAG, "After connect");
                getPatchInfo(internalHandle);
                Log.e(TAG, "After getPatchInfo");
                byte[] data = readPatchFram(internalHandle, 0 , 0x158);
                internalHandle.close();
                Log.e(TAG, "Writing to file " + file_num + ", size = " + data.length);
                
                FileOutputStream f = new FileOutputStream(new File("/data/data/com.eveningoutpost.dexdrip/files/scan"+ file_num + ".dat"));
                NFCReaderX.file_num++;
                f.write(data);
                f.flush();
                f.close();
            }
            catch (IOException e) {
                Log.e(TAG, "Cought exception on doInBackground - readPatchFram failed", e);
            }
            return tag;
            
        }
        
        public byte[] getPatchInfo(NfcV handle) {
            byte[] bArr = null;
                byte[] response = tranceiveWithRetries(handle, new byte[]{(byte) 2, (byte) -95, (byte) 7, (byte) -62, (byte) -83, (byte) 117, (byte) 33});
                if (responseIsSuccess(response)) {
                    bArr = Arrays.copyOfRange(response, 1, response.length);
                }
            return bArr;
        }
        
        private byte[] readPatchFram(NfcV handle, int startAddress, int numberOfBytes) throws IOException {
            int startBlockOffset = startAddress % 8;
            int blockStartAddress = startAddress - startBlockOffset;
            int bytesToRead = numberOfBytes + startBlockOffset;
            if (bytesToRead % 8 != 0) {
                bytesToRead += 8 - (bytesToRead % 8);
            }
            int firstBlock = blockStartAddress / 8;
            byte[] paddedArray = new byte[bytesToRead];
            int blocksToRead = bytesToRead / 8;
            int blockIndex = 0;
            while (blockIndex < blocksToRead) {
                int blockNumber = firstBlock + blockIndex;
                int multiblockReadNumberBlocks = Math.min(3, blocksToRead - blockIndex);
                byte[] response = tranceiveWithRetries(handle, blockNumber <= 255 ? new byte[]{(byte) 2, (byte) 35, (byte) blockNumber, (byte) (multiblockReadNumberBlocks - 1)} : new byte[]{(byte) 2, (byte) -77, (byte) 7, (byte) blockNumber, (byte) (blockNumber >> 8), (byte) (multiblockReadNumberBlocks - 1)});
                if (!responseIsSuccess(response) || response.length < (multiblockReadNumberBlocks * 8) + 1) {
                    return null;
                }
                for (int b = 0; b < multiblockReadNumberBlocks; b++) {
                    for (int i = 0; i < 8; i++) {
                        paddedArray[(blockIndex * 8) + i] = response[((b * 8) + i) + 1];
                    }
                    blockIndex++;
                }
            }
            return Arrays.copyOfRange(paddedArray, startBlockOffset, startBlockOffset + numberOfBytes);
        }
        
        private static boolean responseIsSuccess(byte[] response) {
            return response != null && response.length > 0 && (response[0] & 1) == 0;
        }
        
        // better handle exceptions
        private byte[] tranceiveWithRetries(NfcV handle, byte[] command) {
            byte[] response = null;
            int i = 0;
            while (i < 3) {
                    try {
                        response = handle.transceive(command);
                    } catch (IOException e) {
                        Log.e(TAG, "Cought exception on tranceiveWithRetries - retrying");
                    }
                    if (responseIsSuccess(response)) {
                        return response;
                    }
                    i++;
            }
            return response;
        }
        
```



