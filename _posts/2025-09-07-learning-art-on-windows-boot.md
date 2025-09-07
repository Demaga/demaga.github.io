My machine is fine, but it's not a racing car. When I boot into Windows, there is this noticeable delay between the wallpaper becoming visible, and OS becoming usable. In other words, I stare at my wallpaper for 5 seconds while waiting for Steam to load. So, instead of decluttering the system or upgrading the hardware, I decided to make my wallpaper more interesting to make waiting more enjoyable!

So here is the result:

![Allegory of the Medici family by Cherubino Alberti](/assets/art-wallpaper-medici.png)

# Sourcing data

I went straight to MET looking for an API of some sort. [It does exist](https://metmuseum.github.io/), but I couldn't figure out a way to get a 'random' object. For example, if the api was structured like this: `/api/department_B/id_7`, and I knew how many object there were for every department, I could choose the department on random, and then access random object in range 1-{# of objects in this dep}. 

Thankfully, MET also publishes [an entire dataset](https://www.kaggle.com/datasets/metmuseum/the-met) of public domain art works.

Then I searched for a few other options, and settled on [Victoria and Albert Museum API](https://developers.vam.ac.uk/guide/v2/quick-start.html) because of the [very handly random parameter in /search endpoint](https://developers.vam.ac.uk/guide/v2/restriction/miscellaneous.html?highlight=random#random-random) that made it very easy to integrate this source.

# Creating wallpaper

Basic image manipulation is done by Pillow, no surprises there. For generating caption on the bottom, I used Anthropic API. Didn't really put much thought into it, since any decent LLM would handle this task easily. With 3.5 Sonnet, I hit the sweet spot of price/quality and I'm yet to run out of $5 wortth of credits that I purchased 9 months ago.

# Running script on startup

Now we've come to the only tricky bit here. Generating wallpaper takes some time, with all the API calls and so on. I don't want to do this on startup, since all those 5 seconds of waiting will be spent on waiting for script to finish. Therefore, there are actually two scripts: one for generating wallpaper, and the other just for replacing it.

Replacing wallpaper is a single Windows API call. In Python that would be:
``` python
import ctypes
ctypes.windll.user32.SystemParametersInfoW(20, 0, abs_path, 0)
```

Parameters here might seem mysterious, but [Win32 API](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-systemparametersinfow) explains them pretty well. 20 here refers to `SPI_SETDESKWALLPAPER` command that we need.

To schedule the generating script, we can bundle it as a one-file executable file (.exe) with [PyInstaller](https://pyinstaller.org/en/stable/) and just put it in `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup`. On startup, Windows comes to this folder and just executes [start command](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/start) for every file there. That's it.

For replacing script, we could do the same, but why? It doesn't have any dependencies, and we already have Python installed on the system, so we can just put this `.py` file in the same Startup folder. Well, kind of, but not really. Remember that Windows call `start` command for every file in the Startup directory. When it tries to call `start` on `.py` file, it will call a 'default app' to handle it. So it will only work as a script if 'Python' executable is a default app for your `.py` files, which is probably not the case. It is more likely [IDLE](https://docs.python.org/3/library/idle.html) or your IDE, or Notebook app is the default one. So it won't work like this.

Instead, we could create a `.bat` file, or, like I did, a `.vbs` file, that executes `python ./path/to/replacing_script.py`, and then put that file in Startup directory. Since for that file `start` command will actually start execution, everything will work.

# Speeding things up

After rebooting my system for a test run, I noticed that it still takes more than a second to replace a wallpaper. Why? It's a very simple operation. To be honest, I never found out why, but I guess that Windows just does a bunch of stuff between system boot and calling `start` command on files in Startup folder. 

To fix this, I tried to schedule the same script (replacing one) with [Windows Task Scheduler](https://learn.microsoft.com/en-us/windows/win32/taskschd/task-scheduler-start-page). Thankfully, it does support 'log on' event (up until this point I always referred to it as boot, but it's not an actual system boot we're talking about, but rather me logging into the system). When configuring a task, Action=Start a program refers to the same `start` command we saw earlier, and we just need to pass a path to a `.vbs` script as an argument.

# Conclusion

It's awesome that entities like MET and VAM exist! Thanks for providing that valuable data for free.

Windows has it's quirks, but you get used to it. I didn't encounter anything too insane, and in the end achieved my goal.

Code available here: https://github.com/Demaga/art-wallpaper-changer