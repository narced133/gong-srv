gong-srv
========

System for using an MR2301A WiFi AP (equivalent to Fonera 2100) to control a servo.

Building Images
---------------

This more or less follows the procedure outlined by the OpenWRT [docs](http://wiki.openwrt.org/doc/howto/build).  Note that we insert some patches to customize the default configuration and download a very specific revision.  These instructions were written with a 64-bit Ubuntu 12.04 Server VM for the build.

1. Checkout OpenWRT revision 29592.

        svn co svn://svn.openwrt.org/openwrt/trunk@29592

1. Install build dependencies

        sudo apt-get install build-essential subversion libncurses5-dev zlib1g-dev gawk gcc-multilib flex git-core gettext

1. Place the files in patches/ into root checkout directory.

1. Edit your feeds.conf to add this repo and the following mosquitto dependency. Add it at the top to ensure it takes priority.

        src-git gongsrv git://github.com/narced133/gong-srv.git

1. Download feeds

        ./scripts/feeds update -a
        ./scripts/feeds/install -p owrt_pub_feeds mosquitto
        ./scripts/feeds/install -p gongsrv -a

1. In menuconfig you need to add packages for pitcher, batter and the various gpio packages.  In kernel_menuconfig, you need to add the gpio packages.

        make menuconfig
        make tools/quilt/install
        make kernel_menuconfig
        make defconfig
        make -j 3 # Set to <your number of CPU cores + 1>

1. Flash the images from the /bin directory.

Controlling a servo with batter
------------------------------

You need to export the GPIO and configure the pwm:

    echo 3 > /sys/class/gpio/export
    insmod pwm-gpio-custom bus0=0,1

There's some other nice servo configuration in `/etc/config/batter`.  Otherwise just run it!

Controlling a servo manually
----------------------------

    # Set GPIO numbers
    export G_PWR=3 # Servo power
    export G_SW=7 # Trigger switch (unused in this case)
    export G_CTL=1 # PWM control line

    # Export GPIOs and set direction
    echo $G_PWR > /sys/class/gpio/export
    echo $G_SW > /sys/class/gpio/export
    echo out > /sys/class/gpio/gpio$G_PWR/direction
    echo in > /sys/class/gpio/gpio$G_SW/direction

    # Create the PWM device and configure parameters
    insmod pwm-gpio-custom bus0=0,$G_CTL
    cat /sys/class/pwm/gpio_pwm.0\:0/request # Start ticking
    echo 20000000 > /sys/class/pwm/gpio_pwm.0\:0/period_ns # 20ms period (50hz)
    echo 1200000 > /sys/class/pwm/gpio_pwm.0\:0/duty_ns # 1.2ms ON period
    echo 1 > /sys/class/pwm/gpio_pwm.0\:0/polarity # Make duty cycles more consistent with servo docs

    # Start PWM and configure parameters
    echo 1 > /sys/class/pwm/gpio_pwm.0\:0/run
    echo 1 > /sys/class/gpio/gpio$G_PWR/value

    # Move across the servo range
    echo 700000 > /sys/class/pwm/gpio_pwm.0\:0/duty_ns
    sleep 2
    echo 1200000 > /sys/class/pwm/gpio_pwm.0\:0/duty_ns
    sleep 2
    echo 2100000 > /sys/class/pwm/gpio_pwm.0\:0/duty_ns
    sleep 2
    echo 1200000 > /sys/class/pwm/gpio_pwm.0\:0/duty_ns
    sleep 1

    # Turn the power off
    echo 0 > /sys/class/gpio/gpio$G_PWR/value
    echo 0 > /sys/class/pwm/gpio_pwm.0\:0/run
