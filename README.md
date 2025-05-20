Pihole PiOLED 2025 Guide  -DRAFT-

I recently decided to finally get pihole up and running at home.  When looking at buying two raspberry pi W 2 devices I saw the instructions on how to set up pihole with the PiOLED display on Adafruit's Website.

http://learn.adafruit.com/pi-hole-ad-blocker-with-pi-zero-w

The article shows a last update in 2024 which I incorrectly assumed meant that I'd be able to follow it with no issues in 2025.  I ran into quite a few issues and had to tweak a few things which I'll outline below.

The biggest issue revolves around the fact that Pihole changed the way the API works, specifically the authentication is no longer key based, it's session based and they also changed how the API functions, t

the change to how python opperates in the Bookworm versions of Raspberry Pi OS.  Python scripts are now seperated into Virtual Environments (venv).  This is mentioned in the article but isn't stressed that you need to understand what that means.
For those of us that aren't programmers and looking for true step by step directions it's pretty easy to get a little turned around.  

I also had issues getting the PiOLED to work.

Follow the directions in the wiki section to skip all the headaches I ran into.

