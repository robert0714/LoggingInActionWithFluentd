# LoggingInAction
This contains all the configurations for the examples in the Manning book [Logging In Action with Fluentd, Kubernetes and more](https://www.manning.com/books/logging-in-action?a_aid=Phil)

The contents include:
- Various examples of using Fluentd features
- Worked example of a Fluentd custom plugins
- Scripts and utilities to help accelerate the running of various exercises in the book

For more information check my [blog](https://blog.mp3monster.org/publication-contributions/fluentd-unified-logging-with/) for more information, useful support resources etc.

# Fluentd Documentation
refernece:  https://docs.fluentd.org/

## Install bu Ruby Gem
refernece: 
* https://freecontent.manning.com/deployment-of-fluentd/
* https://rubygems.org/gems/fluentd
* https://www.youtube.com/watch?v=Gp0-7oVOtPw

### Simple deployment of Fuent-bit
https://docs.fluentbit.io/manual/installation/getting-started-with-fluent-bit

### Simple deployment of Fluentd
To get ready to run Fluentd we need to first install Ruby. This is best done by using the latest stable version of Ruby using your operating system’s package framework.  Links to the different installation packages can be found via www.ruby-lang.org. For Windows, we do this by going to the Downloads page has links to the relevant artefact. For Windows, we get taken to ``https://rubyinstaller.org``   to retrieve the Ruby Installer which has been produced to work with a Windows installation tool.

Once downloaded, run the installer, it takes you through the steps to define the preferred location, and it also asks you if you want to install Mysys –say yes to this. Mysys is needed for Ruby Gems, which have a low-level C dependency, such as plugins that call interact with the OS. With it comes several development-related tools such as MinGW, which allows Ruby development to make use of Windows C native libraries. This means we should have Mysys and we recommend taking the full installation with Mingw to support any possible development.

The installer should add Ruby to the Windows PATH environment variable. This can be checked using a Windows shell using the command:
```cmd
echo %PATH%
```

This displays the PATH environment variable which includes the \bin folder of the installation location of your Ruby installation, for example, ``C:\Ruby27-x64``. If it doesn’t appear, it needs to be, and we can add it within the Windows shell with the command:

```cmd
setx path "%path%;c:\dir1\dir2"
```

The c:\dir1\dir2 obviously needs to be replaced with the full path to the bin folder of your Ruby install. For example, ``C:\Ruby27-x64\bin``. Linux obviously has its equivalent settings.

With a fresh Windows shell, it should be possible to execute the command ``ruby –version`` and Ruby displays the installed version. The next step is to get Fluentd installed.

Fluentd can be installed in a variety of different ways.  Treasure Data provide a Windows installer for Fluentd; it introduces a prefix of td into file and folder names. To avoid those confusions, if you take configuration from Windows development environment to a Linux production environment, we aren’t going to use this route, but rather install Fluentd using Ruby Gems, ensuring we don’t get any of these differences. The process of installing via gems is incredibly easy, and with Ruby in our environment path we need only issue the command:
```bash
gem install fluentd
```
As long as you’ve connectivity to ``https://rubygems.org/`` then relevant Gems including dependencies safely download and install. In enterprise and production environments these sites may need to be accessed via a proxy tier.  The installation can be tested by running the following command:

```bash
fluentd  --help
```

This displays the help information for Fluentd.  It should also be possible to see the Fluentd and other gems installed in the deployment location`` lib\ruby\gems\2.7.0\gems\``.
