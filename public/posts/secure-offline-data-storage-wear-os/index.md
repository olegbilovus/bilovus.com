# Secure Offline Data Storage on Wear OS Using Steganography


In emergency situations, access to important data such as digital IDs, passwords, 2FA codes, and emergency contacts may be critical. While smartphones typically store this information, they may not always be accessible. Smartwatches, particularly those running Wear OS, offer an alternative storage option that remains physically accessible.

While encrypted cloud backups provide data redundancy, a local copy stored on a wearable device can offer immediate access during emergencies.

Combining steganography with encryption provides a secure method for storing data on Wear OS devices. This approach embeds encrypted data within an audio file, which can then be stored on the smartwatch. Wear OS does not provide direct file system access through a native "Files" app like Android phones. However, _ADB (Android Debug Bridge)_ can be used to transfer files to and from the smartwatch.

Periodic updates to the stored data can be accomplished directly from an Android phone without requiring a computer. This can be achieved using two apps:

1. [**Termux**](https://f-droid.org/packages/com.termux/): A terminal emulator for Android that enables the use of steganography and encryption tools directly on the phone.
2. [**ADB Shell - Debug Toolbox**](https://play.google.com/store/apps/details?id=com.github.standardadb): An app that facilitates file transfers between the phone and smartwatch using ADB commands through a graphical user interface. Some features may require the pro version.

At the end of this guide, in [**Security Considerations**](#security-considerations), there are important points to consider regarding the security of this method.

## Steganography and Encryption Tools

The steganography tool used is [**Steghide**](http://steghide.sourceforge.net/), a command-line utility that embeds data within audio files. For encryption, [**Openssl**](https://www.openssl.org/) is used, a widely adopted tool for secure data encryption.

To install these tools in Termux, run the following commands:

```bash
pkg update
pkg install steghide openssl
```

## Choosing the Audio File

A WAV audio file serves as an appropriate carrier for the data. WAV files are uncompressed and provide sufficient space to hide encrypted data without significantly altering audio quality. Long-form audio content, such as music compilations of at least one hour, provides ample capacity for data storage.

The audio file can be transferred from phone storage to the Termux home directory. Selecting the file and choosing Termux from the share menu will automatically copy it to the Termux downloads folder at `/data/data/com.termux/files/home/storage/downloads/`.

## Encrypting and Embedding Data

Once the audio file and the data file (e.g., `data.txt`) are in Termux, the following commands can be used to encrypt and embed the data:

```bash
# Encrypt the data file
openssl enc -aes-256-cbc -salt -pbkdf2 -in data.txt -out data.enc -pass pass:your_password

# Embed the encrypted data into the audio file
steghide embed -cf your_audio.wav -ef data.enc

# You will be prompted to set a passphrase for steghide
```

Replace `your_password` with a password of your choice. The `data.txt` file contains the sensitive information you wish to store securely.

The openssl encryption could be skipped if the steghide passphrase is deemed sufficient for the security requirements.

The data does not have to be in a text file; it can be any file type, such as images, PDFs, or an archive containing multiple files.

## Decrypting and Extracting Data

Before transferring the audio file to the smartwatch, it is advisable to test the extraction and decryption process in Termux:

```bash
# Extract the embedded data from the audio file
steghide extract -sf your_audio.wav
# You will be prompted for the steghide passphrase

# Decrypt the extracted data file
openssl enc -d -aes-256-cbc -pbkdf2 -in data.enc
# You will be prompted for the openssl password
```

_Steghide_ will extract the embedded file as `data.enc`, which can then be decrypted to retrieve the original data. It will use the name of the original file when it was embedded. If you gave it a different name, it will use that.

Check that the decrypted file matches the original data file to ensure the process works correctly.

The encrypted data in the audio file will persist until it is overwritten or the audio file is deleted, the extraction process will not remove it from the audio file.

Once we have confirmed that the process works, we can proceed to move/copy the audio file outside of Termux to prepare for transfer to the smartwatch. Simply use the `cp` command to copy the audio file from Termux storage to phone storage:

```bash
cp your_audio.wav /data/data/com.termux/files/home/storage/shared/Music/
```

This copies the audio file to the phone's Music folder, making the file accessible for transfer to the smartwatch.

## Transferring the Audio File to the Smartwatch

Before transferring the audio file to the smartwatch, ensure that:

1. Developer options are enabled on the smartwatch.
2. Wireless debugging (ADB over Wi-Fi) is enabled on the smartwatch.
3. The smartwatch and the phone are connected to the same Wi-Fi network.

Using the _ADB Shell - Debug Toolbox app_, the first step is to connect to the smartwatch. The process is similar to connecting ADB over Wi-Fi from a computer. If a local VPN for DNS filtering is active on the phone, it may need to be disabled temporarily to allow the connection. The connection may require several attempts to succeed. After a long period of inactivity (days), it may be necessary to unpair and re-pair the devices from the smartwatch and phone settings.

If connection errors occur, verify that the _Connection Port_ in the app matches the port shown on the smartwatch under _Wireless Debugging_. The port changes after each connection, so it may need to be updated in the app.

When the connection is successful, you will see the smartwatch listed as a connected device in the app as shown below:

{{< center_image src="/images/posts/secure-offline-data-storage-wear-os/connected.jpg" alt="ADB Connected to Smartwatch" width="250px" >}}

Once connected, scroll the upper toolbar horizontally to find the "Local files" option. From there:

1. Navigate to the Music folder and select the audio file options menu (three vertical dots) and choose "Push file".
2. Set a destination directory on the smartwatch. In this example, `/sdcard/Music/Samsung/` is used.
3. After setting the directory, repeat steps 1 and 2 to push the file. A toast notification will indicate that the file is being transferred.
4. After a few seconds, the file should be transferred to the smartwatch and you will see a success toast notification.
5. Go to the "Device files" option in the app to verify that the file is present in the specified directory on the smartwatch. If the file is not visible, refresh the view by pulling down or re-enter the directory.

Below are the screenshots of each step in the process:

{{< images_row src="/images/posts/secure-offline-data-storage-wear-os/step0.jpg,/images/posts/secure-offline-data-storage-wear-os/step1.jpg,/images/posts/secure-offline-data-storage-wear-os/step2.jpg,/images/posts/secure-offline-data-storage-wear-os/step3.jpg,/images/posts/secure-offline-data-storage-wear-os/step4.jpg,/images/posts/secure-offline-data-storage-wear-os/step5.jpg" captions="Step 0: Local Files|Step 1: Select Audio File|Step 2: Set Destination Directory|Step 3: Push File|Step 4: Confirm File Transfer|Step 5: Verify File on Smartwatch" width="250px" >}}

## Transferring the Audio File Back to the Phone

To retrieve the audio file from the smartwatch, connect to the smartwatch using the ADB Shell - Debug Toolbox app and navigate to the directory containing the audio file using the "Device files" option. Select the audio file options menu (three vertical dots) and choose "Pull file". Set a destination directory on the phone. A toast notification will indicate the transfer progress, followed by a success notification when complete.

The decryption and extraction process can then be performed in Termux as described earlier in the [Decrypting and Extracting Data](#decrypting-and-extracting-data) section.

## Security Considerations

If the smartwatch is lost or stolen, an attacker with physical access to the device could potentially extract the audio file using ADB commands. However, without the steghide passphrase and the openssl password (if used), the data remains protected.

Choose a steghide and openssl passphrase that you can remember in an emergency. If you're a high risk profile, use a stronger passphrase. If not, pick one that's easy to recall when you don't have access to your password manager.

Even if the smartwatch is lost or stolen, attackers are unlikely to examine audio files for hidden data unless specifically targeting the owner. The use of steganography adds an additional layer of obscurity, making it less likely that the presence of sensitive data would be detected. Stolen smartwatches are typically reset to factory settings and resold rather than examined for hidden data.


---

> Author: [Oleg Bilovus](https://bilovus.com/about/)  
> URL: https://bilovus.com/posts/secure-offline-data-storage-wear-os/  

