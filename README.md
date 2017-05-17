# THETA S WiFi Live Streaming to VR Headsets

This Unity project (created in the version 5.6.1) has as main goal to live stream a 360°-video acquired with the [THETA S](https://theta360.com/en/about/theta/s.html) camera to a VR headset.

For the moment, only the [Oculus Rift](https://www.oculus.com/rift/) is supported; the **Oculus Development Kit 2 (DK2)** was utilized for test. Support for the [Google Cardboard](https://vr.google.com/cardboard/) will be implemented in the future.

This project is based on the [ThetaWifiStreaming repository](https://github.com/makoto-unity/ThetaWifiStreaming) by [Makoto Ito](https://github.com/makoto-unity). The original `Assets`, `Scripts` and `LICENSE` files are located in [`./Assets/ThetaStreaming/`](https://github.com/rcassani/ThetaWifiStreaming2VRheadset/tree/master/Assets/ThetaStreaming)    

[Live Streaming from the RICOH THETA S ](http://theta360.guide/community-document/live-streaming.html)
can be performed through the USB or HDMI cables. While the THETA S is not meant for live-streaming over WiFi, a small [Equirectangular](http://theta360.guide/blog/video/2017/01/25/convert-dual-fisheye-to-equirectangular.html) preview (MotionJPEG 640x320 @ 10 fps) is streamed over Wifi, as guidance in the image capture (shooting still images) mode.

## Requirements
Run the [Rift's setup tool](https://www3.oculus.com/en-us/setup/) to install all the required software and configure the Oculus Rift.

## Usage
1. Turn the THETA S on in image capture (shooting still images) mode with WiFi enabled.
  https://theta360.com/en/support/manual/s/content/shooting-photo/shooting_photo_02.html

2. Establish the WiFi connection between the PC and the THETA S. The password is written on the THETA S, the 8 characters after XS (not including XS)

3. Run Oculus Windows App in the background

4. Open the `ThetaWifiStreamingOculus` scene of this project, and click in the Play button.

5. By default, the front orientation of the Oculus Rift corresponds to the front lens (the one without shooter button) of the THETA S. To reset this orientation, press the `SPACE` bar, the current orientation of the Oculus Rift will correspond to the front lens.

## Actions during execution
During the "game" the user can change the Scene with the following keys

| Key    | Action                           |
| -------|:--------------------------------:|
| `ESC`  | Stops the WiFi streaming,        |
| `SPACE`| Resets Oculus orientation        |
| `W`    | Increases the size of the Sphere |
| `S`    | Reduces  the size of the Sphere  |

With a small Sphere size, for example 20, the movement of the head (relative to the head tracking sensor) has  a zoom in (or out) effect when the Oculus Rift gets closer (or farther) to the sensor, as in the Unity scene the Oculus Camera position gets closer (or farther) to the spherical projection. With Sphere size values, for example 220, the movement of the head is negligible.

The translation and original `README.md` of the repo are included below.
___

# ThetaWifiStreaming
Theta S Wifi Streaming Demo with Unity

## Things to Prepare
1. Theta S
2. A PC or Mac that is connected to Wifi. iOS or Android will probably work as well.

## Usage
1. Turn the THETA S on with Wifi mode enabled.
2. Establish the WiFi connection between your PC/Mac and the THETA. The password is written on the THETA, the 8 characters after XS (not including XS)
3. When you open the project in Unity, the streaming image will appear on the material of Sphere100

## Acknowledgements
- I referenced Noshipu's blog. Thank you.  http://noshipu.hateblo.jp/entry/2016/04/21/183439
- This project uses LitJson to read Json format. https://lbv.github.io/litjson/ Thank you very much.

## Translation of Noshipu's blog

This article was originally written by [@noshipu](https://twitter.com/noshipu),
CEO of [ViRD, Inc](http://vird.co.jp/).

![](http://theta360.guide/blog/img/2016-10/unity1.png)

![](http://theta360.guide/blog/img/2016-10/unity-middle.png)

![](http://theta360.guide/blog/img/2016-10/unity2.png)



# About the RICOH THETA API

In order to use Wifi live streaming, you must use the `_getLivePreview` API.
[Official Reference](https://developers.theta360.com/en/docs/v2.0/api_reference/commands/camera._get_live_preview.html)

> NOTE from Craig: This was replaced by [getLivePreview](https://developers.theta360.com/en/docs/v2.1/api_reference/commands/camera.get_live_preview.html) in version 2.1 of the API. This blog by Noshipu-san refers to the 2.0 API, which is still supported by
the THETA S. Be aware of the differences in your code.

Unlike the other APIs, `_getLivePreview` is different because the data is in a stream and keeps going. You will not be able to get a WWW class to wait until the request is complete (maybe).

> NOTE from Craig: This is the major problem developers have when working with `getLivePreview`. As the data
> is a stream, you can't want for the data to end before running your next command. For example, it's
> different from downloading and displaying an image or video file because you know when the transfer is
> complete.

# Processing Flow

## Set the POST request to create a HttpWebRequest class

    string url = "Enter HTTP path of THETA here";
    var request = HttpWebRequest.Create (url);
    HttpWebResponse response = null;
    request.Method = "POST";
    request.Timeout = (int) (30 * 10000f); // to ensure  no timeout
    request.ContentType = "application/json; charset = utf-8";

    byte [] postBytes = Encoding.Default.GetBytes ( "Put the JSON data here");
    request.ContentLength = postBytes.Length;

## Generate a class of BinaryReader to get the byte data (you get the bytes one by one)

    // The start of transmission of the post data
    Stream reqStream = request.GetRequestStream ();
    reqStream.Write (postBytes, 0, postBytes.Length) ;
    reqStream.Close ();
    stream = request.GetResponse () .GetResponseStream ();

    BinaryReader reader = new BinaryReader (new BufferedStream (stream), new System.Text.ASCIIEncoding ());

## Get the start and stop bytes of 1 frame of the MotionJPEG and cut out one frame

With the byte, check the partion value of the MotionJPEG.

    ...(http)
    0xFF 0xD8      --|
    [jpeg data]      |--1 frame of MotionJPEG
    0xFF 0xD9      --|
    ...(http)
    0xFF 0xD8      --|
    [jpeg data]      |--1 frame of MotionJPEG
    0xFF 0xD9      --|
    ...(http)

Please refer this answer on StackOverflow to
[How to Parse MJPEG HTTP stream from IP camera?](http://stackoverflow.com/questions/21702477/how-to-parse-mjpeg-http-stream-from-ip-camera)  

The starting 2 bytes are `0xFF, 0xD8`. The end bye is `0xD9`

The code is shown below.

    List<byte> imageBytes = new List<byte> ();
    bool isLoadStart = false; // 画像の頭のバイナリとったかフラグ
    byte oldByte = 0; // 1つ前のByteデータを格納する
    while( true ) {
        byte byteData = reader.ReadByte ();

        if (!isLoadStart) {
            if (oldByte == 0xFF){
                // First binary image
               imageBytes.Add(0xFF);
            }
            if (byteData == 0xD8){
               // Second binary image
               imageBytes.Add(0xD8);

               // I took the head of the image up to the end binary
               isLoadStart = true;
            }
        } else {
            // Put the image binaries into an array
            imageBytes.Add(byteData);

            // if the byte was the end byte
            // 0xFF -> 0xD9 case、end byte
            if(oldByte == 0xFF && byteData == 0xD9){
                // As this is the end byte
                // we'll generate the image from the data and can create the texture
                // imageBytes are used to reflect the texture
                // imageBytes are left empty
                // the loop returns the binary image head
                isLoadStart = false;
            }
        }
        oldByte = byteData;
    }

## Texture Generation Separated by Byte

This is the byte to reflect the texture.

    mainTexture.LoadImage ((byte[])imageBytes.ToArray ());

# ThetaWifiStreaming
Theta S Wifi Streaming Demo with Unity

## 準備するもの
1. Theta S
2. Wifiが使えるPC/Mac（iOS/Androidでも多分使えます）

## 使い方
1. Theta S を起動させて、Wifiモードをオンにする。
2. PC/Mac と Theta をWifiで繋げる。パスワードはThetaの底に書いてある、「XS」に後に続く数字８桁（XSは除く）。
3. このプロジェクトをUnityで開いてスタートさせると、Sphere100のマテリアルにストリーミング画像が出る。

## 謝辞
- のしぷさんのブログを参考にさせていただきました。 http://noshipu.hateblo.jp/entry/2016/04/21/183439 ありがとうございました！
- This project uses LitJson to read Json format. https://lbv.github.io/litjson/ Thank you very much.

![Analytics](https://ga-beacon.appspot.com/UA-73311422-5/ThetaWiFiStreaming)
