# FRC Specific Resources

This is a guide of resources for FRC programming, and some other useful stuff. 

## WPIlib 

Installing WPIlib - https://docs.wpilib.org/en/stable/docs/zero-to-robot/step-2/wpilib-setup.html 
WPIlib Documentation - https://docs.wpilib.org/en/stable/docs/zero-to-robot/introduction.html 

WPIlib downloads (almost) everything you would need for FRC programming in WPIlib are included 
- Advantage Scope(and AdvantageKit), which allows you to log, simulate and replay robot competitions, and actively monitor data during testing. 
- VS Code, which includes a suite of WPIlib tools like `Build Robot Code`, `Deploy Robot Code` and `Simulate Robot Code` being the most widely used.
!!! note
    VS Code (Microsoft Visual Studio Code) is the supported IDE for FRC - you can use something else but it will make your work more difficult. Also if you're doing any personal coding projects in VS Code it is recommended to download VS Code from https://code.visualstudio.com/, because extensions you might install can mess with WPIlib,
- Elastic, software used for creating a dashboard that is used by the Drive-Team in competition.
- Also, WPIlib downloads C++, Python and Java compilers onto your device - we use Java, but you can see these in VS Code as downloaded extensions.
!!! warning
    Git is not downloaded with WPIlib, but you will need it, download it here https://git-scm.com/downloads, or through terminal, more on Git Later.

## Pathplanner

Installing Pathplanner - https://github.com/mjansen4857/pathplanner/releases 
Pathplanner Documentation - https://pathplanner.dev/home.html 

- Pathplanner is a tool for creating Paths and Autos for Autonomous Sequences.

## Git and Github

Installing Git - https://git-scm.com/downloads 

- Git is used to get code you wrote locally on your computer to somewhere it's hosted, in our case Github (There are alternatives like Gitlab or Self-Hosting)
- You can use the UI in VS Code or, the command line 
- Git works like a tree where branches are created off of main (the trunk of the tree)

`git pull` - pulls from the current version (the version on Github) of the branch you are on. 
`git push` - pushes your changes to your branch making them viewable on github.

### Github

- Github is what we use to host our code
- Creating a branch means that you have access to code written in main (or master), and you can pull from main to get updated to the current version
- A PR or Pull Request, is a request to get your  changes merged into main (or master)





