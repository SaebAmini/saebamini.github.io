---
layout: post
title: Remove Your Conference Call Background Noise for Free!
---

As working from home has become the new normal for a lot of professionals during the pandemic, communication over conference calls has posed more of a challenge. I'm not talking about the [funny WFH incidents](https://youtu.be/rOWGe7uOuPU) as people and their families are adapting to the new situation. I'm talking about not being able to leverage non-verbal communication as before, which is commonly believed to be responsible for the majority of our communication's impact.

With fewer communication opportunities and negligible body language visibility, we're now relying on the clarity of our words and voice more than ever. Having a decent sound setup is no longer just a nice-to-have for those occasional WFH days – it's become essential.

Having [a decent microphone](https://www.bluemic.com/en-us/products/yeti-nano/) is a good start, but improving your setup doesn't necessarily have to cost much. In fact, I recently improved my setup a great deal without spending any money by just removing background noise using free software. Let me take you through how I did it:<!--more-->

## Walkthrough

1. What makes most of this possible is the amazing (and open-source) [OBS Studio](https://obsproject.com/) which is popular with streamers. Go ahead and install it. While you can do many fancy things with it, we're going to use its [audio filters](https://obsproject.com/wiki/Filters-Guide#audio-device-filters) for our purpose.

2. Another core part of the setup is [Virtual Audio Cable](https://www.vb-audio.com/Cable/), which is also free. While there are [OBS plugins available to provide its video as a virtual webcam](https://obsproject.com/forum/resources/obs-virtualcam.949/), there are no equivalent plugins for sound yet. We're going to use this software to capture the filtered/noiseless OBS audio output and provide it as a virtual microphone that we can use in conferencing software like Teams and Zoom. Go ahead and install this one as well.

3. In OBS Studio's Audio Mixer, click the settings cog icon next to Mic/Aux and choose `Advanced Audio Properties`:

   ![OBS Mic/Aux Advanced Audio Settings](/images/posts/remove-bg-noise/1.png)

   Then set `Audio Monitoring` for your Mic/Aux to `Monitor and Output`:

   ![Audio Monitoring](/images/posts/remove-bg-noise/2.png)

4. Go to the Audio settings tab from `File → Settings → Audio` and under `Advanced`, set `Monitoring Device` to `CABLE Input (VB-Audio Virtual Cable)`:

   ![Monitoring Device](/images/posts/remove-bg-noise/3.png)

   This sends the filtered OBS audio output to the virtual input, which in turn directs it to `CABLE Output (VB-Audio Virtual Cable)`.

5. To make tweaking OBS filters easier, go to sound settings for the `CABLE Output` recording device in your Control Panel and turn on `Listen to this device`:

   ![Listen to this device](/images/posts/remove-bg-noise/4.png)

   This lets you hear the filtered OBS audio output immediately so you can tweak the filters until you're happy with the result.

6. You are now ready to tweak the OBS audio filters to remove your background noise. Same as step 3 above, in OBS Studio's Audio Mixer, click the settings cog icon next to Mic/Aux but this time choose `Filters`:

   ![OBS Audio Filters](/images/posts/remove-bg-noise/5.png)

   There 3 main audio filters here that'll help you remove noise and unnecessary sounds:

   1. [**Noise Suppression**](https://obsproject.com/wiki/Filters-Guide#noise-suppression) can be used to remove mild background noise or white noise that may be in any of your audio sources (e.g. AC/Computer fan). Start at `-10 dB` and move the slider to the left until you don't hear them anymore.
   2. [**Noise Gate**](https://obsproject.com/wiki/Filters-Guide#noise-gate) lets you cut your microphone off when you're not talking. This is helpful when you're in active conversation, otherwise, it's much better to just mute yourself when you're not talking for extended periods. Select a close threshold above the noise volume and an open threshold slightly below your normal talking volume for good results.
   3. [**Expander**](https://obsproject.com/wiki/Filters-Guide#expander) is pretty powerful and lets you reduce background noise such as computer fans, mouse/keyboard clicks, breathing, unwanted mouth noises and faint chatter/music. In short, it makes quiet sounds quieter. You typically should place this near the end of your filter chain, but you can sometimes get pretty good results with correctly configuring this filter alone based on your environment.

   Once you're happy with the results, turn off `Listen to this device` from your sound settings.
   

7. Now in your conferencing software like Teams, Zoom, etc. or any other software that uses a microphone, just set your microphone to `CABLE Output (VB-Audio Virtual Cable)` and everyone will hear the audio that's gone through your OBS filters.

   ![Choose virtual microphone](/images/posts/remove-bg-noise/6.png)



That's it. There's no excuse for you to continue sharing your neighbour's lawnmower sound with everyone on the call!