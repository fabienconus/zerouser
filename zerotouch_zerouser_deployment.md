# Zero-touch, zero-user deployment

## Summary

In this short text, I will explain how we deploy our macOS configuration with JAMF and DEPNotify on the login window, without the need of having a user log in.

## The context

I am a physics and math teacher and a Mac admin in a public school system in Switzerland. We manage approximately 2500 Macs in 45 different schools in secondary education (students being between 12 years old and 18 years old or more).

All students and teachers work in a mixed environment made of computers running macOS, Linux and Windows.

The difficulty we are facing is that unlike in an enterprise setting, none of our Macs are owned or used by a single user. They are all destined to be used by a lot of users with a wide variety of profiles (students, teachers, administrative staff, and so on).

In terms of deployment, this specificity means that we cannot deploy preferences directly in the users home directory, since users will be created during the school year, as they log on to the various computers. We must therefore deploy everything in the User Template.

## The challenge

We hear a lot about zero-touch deployment. The goal being that a user gets his new computer and is up and running as quickly as possible with as limited interaction as possible.

A lot of MDM vendors advertise zero-touch deployment (just google "_insert\_mdm\_name\_here_ zero touch deployment" and you are guaranteed to find a solution). But in our situation we have an additional requirement, we have to have a zero-touch **zero-user** deployment.

We could have created a temporary user that would be deleted after the deployment is complete, but this was not an acceptable solution for us for various security reasons.

## The software

Our IT department has opted for [JAMF](https://www.jamf.com/home-2/) as our MDM, since they also use it to manage all our iPads. 

For the deployment process, we decided to use [DEPNotify](https://gitlab.com/Mactroll/DEPNotify) to provide a graphical interface to what is being installed. DEPNotify is an app that provide a graphical display of the progression of a deployment, and is controlled by writing dynamically to a file. A typical way to use DEPNotify to provide feedback is via a shell script such as the one provided by JAMF: [DEPNotify-Starter](https://github.com/jamf/DEPNotify-Starter)

That said, the choice of software is not really relevant to what will be described below.

## The concept and solution

### Running a graphical user interface on the login window

As I worked on this I realized I had a misconception about LaunchAgents and LaunchDaemons. There are a lot of resources on the internet about these and most of the time, you will read that the bottom line is that LaunchDaemons are processes run as root (i.e. with administrator privileges) typically at startup, while LaunchAgents are run in the user-space, as the user, and typically when the user logs in.

The other difference is that processes executed by LaunchDaemons cannot display a graphical user interface, while LaunchAgents can.

Now we run into why what I wanted to achieve seemed difficult : the DEPNotify-starter script is responsible for launching the DEPNotify application (which has a graphical user interface), while also making calls to the jamf binary to execute policies, and that requires administrative privileges (i.e. it requires to be run as root).

Do I use a LaunchDaemon in order to have administrative privileges ? Or do I use a LaunchAgent in order to be able to display a graphical user interface.

It turns out, that a LaunchAgent that loads on the LoginWindow and is located in `/Library/LaunchAgent` is actually owned by root and can execute scripts as root ! 
The trick to make a LaunchAgent load on the login window consists just in adding this to its plist :

	<key>LimitLoadToSessionType</key>
	<string>LoginWindow</string>

This simple trick will allow a LaunchAgent to be loaded with the loginwindow process.

### DEPNotify

There is however another problem. DEPNotify, as most deployment solutions, require a user to be logged in. As a matter of fact, the [DEPNotify-starter script](https://github.com/jamf/DEPNotify-Starter) has these few lines of code that wait for the Finder to be running :

	# Checking to see if the Finder is running now before continuing. This can help
	# in scenarios where an end user is not configuring the device.
	FINDER_PROCESS=$(pgrep -l "Finder")
	until [ "$FINDER_PROCESS" != "" ]; do
	    echo "$(date "+%a %h %d %H:%M:%S"): Finder process not found. Assuming device is at login screen." >> "$DEP_NOTIFY_DEBUG"
	    sleep 1
	    FINDER_PROCESS=$(pgrep -l "Finder")
	done

You would think that it would be enough to just remove these lines, but no, because in the code for DEPNotify in the `AppDelegate.swift` file you will find this :

	var dockRunning = 0
	let ws = NSWorkspace.shared
	while dockRunning == 0 {
		print("Waiting for the Dock")
		dockRunning = ws.runningApplications.filter({ $0.bundleIdentifier == "com.apple.dock" }).count
		RunLoop.main.run(mode: RunLoop.Mode.default, before: Date.distantFuture)
	}

Which means that even if you tell the script not to wait for the Finder to be running, DEPNotify itself will wait for the Dock to be loaded, which means that it is necessary for a user to log in.

Don't get me wrong, there are tons of good reasons to do this, especially if the computer is destined to be used by a local user. In most enterprise situations, this is definitely to good way to do things, since it ensures all preference files have been created and can therefore be modified by the deployment.

In our case, every preference we deploy will go in the user template, which is why we don't need a user to be logged in.

In order for DEPNotify to be able to run on the login window, with no user logged in, we must delete or comment the two blocks of code shown above in the starter script (if you want to use it) and in the code from DEPNotify.

The second modification we have to make is to allow the DEPNotify window to be visible when no user is logged in. This is done by adding these two lines to the `ViewController.swift` file in the `windowDidLoad()` function, right after `background?.sendBack()` :

	NSApp.windows[0].canBecomeVisibleWithoutLogin = true
	NSApp.windows[0].orderFrontRegardless()

The same must also be done for the Background, to allow for that nice background blur to be visible. These two lines must therefore be added to the `Background.swift` file in the `windowDidLoad()` function, right after `backgroundWindow.setFrameOrigin((NSScreen.main?.frame.origin)!)` :

	backgroundWindow.canBecomeVisibleWithoutLogin = true        
	backgroundWindow.orderFrontRegardless()

Of course, this means that the DEPNotify application will  have to be recompiled, signed and notarized.

### The solution

Using all this, I was able to create my zero-touch, zero-user deployment. This is how it works :

After the enrollment, I run a script that creates a LaunchAgent in `/Library/LaunchAgent` responsible for triggering the DEPNotify starter script. In order for this LaunchAgent to load, I use the `launchctl load -S LoginWindow` command. Here is what it looks like :

	#!/bin/sh
	#
	#  Created by Fabien Conus for the State of Geneva
	#
	launchAgentName="ch.ge.edu.runDEPNotify.plist"
	launchAgentPath="/Library/LaunchAgents/${launchAgentName}"
	
	# Write the LaunchAgent to execute the DEPNotify script
	echo "Writing agent ${launchAgentPath}"
	defaults write ${launchAgentPath} Label ${launchAgentName}
	defaults write ${launchAgentPath} LimitLoadToSessionType "LoginWindow"
	defaults write ${launchAgentPath} ProgramArguments -array-add "/usr/local/bin/jamf"
	defaults write ${launchAgentPath} ProgramArguments -array-add "policy"
	defaults write ${launchAgentPath} ProgramArguments -array-add "-event"
	defaults write ${launchAgentPath} ProgramArguments -array-add "runDEPNotify"
	defaults write ${launchAgentPath} RunAtLoad -bool true
	defaults write ${launchAgentPath} LaunchOnlyOnce -bool true
	defaults write ${launchAgentPath} StandardErrorPath "/tmp/runDEPNotify.err"
	defaults write ${launchAgentPath} StandardOutPath "/tmp/runDEPNotify.out"
	chown root:wheel ${launchAgentPath}
	chmod 644 ${launchAgentPath}
	
	# Load the agent
	launchctl load -S LoginWindow ${launchAgentPath}
	
	exit 0
_Note: you could also package the plist and add the launchctl command as a postinstall script_

The `runDEPNotify` trigger will tell JAMF to execute the policy that contains the modified DEPNotify-starter script, which will in turn execute the modified DEPNotify application.

### Conclusion

With very few changes, it is quite simple to create a deployment process that does not require a user to log in. It has been working for us flawlessly for 2 years now.

I hope this can be useful for other people, and please feel free to [contact me](mailto:fabien.conus@edu.ge.ch) if you need more information.

Finally, I would like to thank Armin Briegel who motivated me to write this and Pico Mitchell who provided the elegant way to load a LaunchAgent without killing the loginwindow process.