.. _common-imu-notch-filtering:

[copywiki destination="copter,plane"]

===========================================================
Managing Gyro Noise with the Dynamic Harmonic Notch Filters
===========================================================

As discussed under the :ref:`Vibration Damping<common-vibration-damping>` topic, managing vibration in ArduPilot autopilot installations is extremely important in order to yield predictable control of an aircraft. Typically installations utilize mechanical vibration damping for the autopilot, internally or externally, in order to remove the worst of the vibration. However, mechanical damping can only go so far and software filtering must be used to remove further noise.

To the autopilot, vibration noise looks like any other disturbance (e.g. wind, turbulence, control link slop, etc.) that the autopilot must compensate for in order to control the aircraft. This prevents optimum tuning of the attitude control loops and decreased performance.

ArduPilot provides two filtering mechanisms for noise. Lowpass filters on the accelerometer signals, controlled by the :ref:`INS_ACCEL_FILTER<INS_ACCEL_FILTER>`, and  the gyro signals, controlled by :ref:`INS_GYRO_FILTER<INS_GYRO_FILTER>`, and Harmonic Notch Filters on the gyro signals.

As discussed in :ref:`common-measuring-vibration` section, there are basically two classes of noise/vibrations: those generated within the bandwidth of the gyros/accelerometer sampling and noise above those frequencies which appear within the bandwidth and can cause the "leans". The aliased noise must be eliminated at the source with improved mounting or frame rigidity, but the above filters can deal with the other.

For multicopters and QuadPlanes, virtually all vibrations originate from the motor's rotational frequency.  For helicopters and planes, the vibrations are linked to the main rotor/prop speed.

ArduPilot has support for two notch filters whose filter frequency can be linked to the motor rotational frequency for motors, or the rotor speed for helicopters, and provides notches at a primary frequency and its harmonics.

While the lowpass filters can effectively diminish the impact of this noise, having low frequency set points creates a lot of phase lag and therefore reduces how aggressive the tune can be before oscillation occurs, which results in a poorer tune.

For the gyro based rate controllers, this reduces their ability to respond to fast disturbances. If the gyro lowpass filter can be set higher, the phase lag induced is lower and the tune can be more aggressive. But this allows more noise and vibration, effectively canceling that gain out. Enabling either one or both of the Harmonic Notch filters allow targeting the noise generated by the motors, allowing a higher frequency of the following low pass to be set and therefore allowing tighter tune.

The harmonic notch is enabled overall by setting :ref:`INS_HNTCH_ENABLE<INS_HNTCH_ENABLE>` = 1 for the first notch, and :ref:`INS_HNTC2_ENABLE<INS_HNTC2_ENABLE>` = 1, for the second. After rebooting, all the relevant parameters will appear.

Various methods of dynamically adjusting the notch(s) center frequency to track motor speed under different thrust conditions is provided, ie Dynamic Harmonic Notch filtering.

Notch Filter Control Types
==========================

Key to the dynamic notch filter operation is control of its center frequency. There are five methods that can be used for doing this:

#. :ref:`INS_HNTCH_MODE <INS_HNTCH_MODE>` = 0. Dynamic notch frequency control disabled. The center frequency is fixed and is static. Often used in Traditional Helicopters with external governors for rotor speed, either incorporated in the ESC or separate for ICE motors.
#. :ref:`INS_HNTCH_MODE <INS_HNTCH_MODE>` = 1. (Default) Throttle position based, where the frequency at hover throttle is determined by analysis of logs, and then variation of throttle position above this is used to track the increase in noise frequency. Note that the throttle reference only applies to VTOL motors in QuadPlanes, not forward motors, and will not be effective in fixed wing only flight modes. See :ref:`throttle-based<common-imu-notch-filtering-throttle-based-setup>` for further setup details.
#. :ref:`INS_HNTCH_MODE <INS_HNTCH_MODE>` = 2 (RPM Sensor 1) or 5(RPM Sensor2). RPM sensor based, where an external :ref:`RPM sensor <common-rpm>` is used to determine the motor frequency and hence primary vibration source's frequency for the notch. Often used in Traditional Helicopters (See :ref:`Helicopters<common-imu-notch-filtering-helicopter-setup>`) using the ArduPilot Head Speed Governor feature. See :ref:`RPM Sensor<common-rpm-based-notch>` for further setup instructions.
#. :ref:`INS_HNTCH_MODE <INS_HNTCH_MODE>` = 3. ESC Telemetry based, where the ESC provides motor RPM information which is used to set the center frequency. This can also be used for the forward motor in fixed wing flight, if the forward motor(s) ESCs report RPM. This requires that your ESCs are configured correctly to support BLHeli telemetry via :ref:`a serial port<blheli32-esc-telemetry>`. See :ref:`ESC Telemetry<common-esc-telem-based-notch>` for further setup instructions. If :ref:`INS_HNTCH_OPTS<INS_HNTCH_OPTS>`, or :ref:`INS_HNTC2_OPTS<INS_HNTC2_OPTS>` if the second set of notches is enabled, has bit 1 set, then a set of notches for each motor will be created, tracking its RPM telemetry, otherwise, the average frequency of all motors will set the center frequency.
#. :ref:`INS_HNTCH_MODE <INS_HNTCH_MODE>` = 4. If your autopilot supports it (ie. has more than 2MB of flash, see :ref:`common-limited_firmware`), In-Flight FFT, where a running FFT is done in flight to determine the primary noise frequency and adjust the notch's center frequency to match. This probably the best mode if the autopilot is capable of running this feature. This mode also works on fixed wing only Planes. See :ref:`In-Flight FFT <common-imu-fft>` for further setup instructions.

All of the above are repeated, independently, for the second notch and are prefaced with ``INS_HNTC2_`` instead of ``INS_HNTCH_``. The following will explain setup for the first set of notches.

.. note:: only one filter can be mode 4(FFT).


Determining Notch Filter Center Frequency
=========================================

Before actually setting up a dynamic notch filter, the frequencies that are desired to be rejected must first be determined. This is crucial if :ref:`common-throttle-based-notch` is used. While the other methods do not require this knowledge apriori, its still worthwhile as a comparison point for the post filter activation analysis of the filter's effectiveness

Once the noise frequency is determined, the notch filter(s) can be further setup. Historically, :ref:`common-imu-batchsampling` has been used for this (also for slow cpu's like F4-based autopilots), logging short bursts of raw IMU data for spectral analysis.

As of firmware version 4.5 and later, a better method has been developed using continuous raw IMU data, if the autopilot is H7-based, and using a new web based tool. :ref:`This method is now preferred <common-raw-imu-logging>`.

Number of Harmonics Filtered
============================

- Set :ref:`INS_HNTCH_HMNCS <INS_HNTCH_HMNCS>` to enable up to three harmonic (multiples of the center frequency) notches. For ESC :ref:`INS_HNTCH_MODE <INS_HNTCH_MODE>` tracking, each motor will get a set of these notches. If an octocopter sets up all three harmonics, this results in 8 x 3 = 24 notch filters. Enabling triple notches (see below) would result in 72 filters! This would most certainly cause excessive cpu loading and performance issues. Other modes only provide a single set of harmonic notches.

Always enable only the number of harmonic notch filters actually required and be especially aware of what is being enabled if using ESC (:ref:`INS_HNTCH_MODE <INS_HNTCH_MODE>` = 3) tracking mode.

FFT Dynamic Harmonic Notch Frequency Tracking
=============================================

FFT mode tracking sets the base frequency to the largest noise peak. Normally, when multiple harmonic notch filters are then enabled, the center frequency of each harmonic is locked to the base frequency of the first filter as an integer multiple, as determined by :ref:`INS_HNTCH_HMNCS <INS_HNTCH_HMNCS>`. Setting bit 1 of :ref:`INS_HNTCH_OPTS<INS_HNTCH_OPTS>`, or :ref:`INS_HNTC2_OPTS<INS_HNTC2_OPTS>`, will enable each harmonic filter to track the largest noise peaks, individually.

.. note:: setting bit 1 of the notch options will also change the default value of :ref:`INS_HNTCH_HMNCS <INS_HNTCH_HMNCS>` to 1 instead of its normal 3. This is to maintain backwards compatibility with previous fimware versions. You can set :ref:`INS_HNTCH_HMNCS <INS_HNTCH_HMNCS>` back to 3, or whatever is desired, after setting bit 1 of :ref:`INS_HNTCH_HMNCS <INS_HNTCH_HMNCS>`.

Checking Notch Filter Effectiveness
===================================

Once the notch filter(s) are setup, the effectiveness of them can be checked by again measuring the  frequency spectrum of the output of the filters (which are the new inputs to the IMU sensors). Refer back to the :ref:`common-imu-batchsampling`  or :ref:`common-raw-imu-logging` page for this.

Multi Notch
===========

The software notch filters used are very "spikey" being relatively narrow but good at attenuation at their center. On larger copters the noise profile of the motors is quite dirty covering a broader range of frequencies than can be covered by a single notch filter. In order to address this situation it is possible to configure the harmonic notches as multiple notches that gives a wider spread of significant attenuation. The configuration is controlled by the :ref:`INS_HNTCH_OPTS <INS_HNTCH_OPTS>` parameter. This is a bitmask parameter and multiple options are possible at the same time, but using bit 0, 1, and bit 4 at the same time should be avoided. Use only one of those in a given configuration.

==========================================      =======================
:ref:`INS_HNTCH_OPTS <INS_HNTCH_OPTS>` Bit      Action
==========================================      =======================
0                                               Double overlapping Notches
1                                               MultiSource: if using FFT Mode, the three largest noise sources will have a notch assigned. If ESC Telemetry Mode, then each motor will have a notch assigned at its RPM.
2                                               Updates the filters at the loop rate. This is cpu intensive, but tracks noise variations faster. Only valid if frequency source updates at loop rate, ie Bi-Directional DShot telemetry.
3                                               Enables notches on every IMU instead of just the primary. This is cpu intensive, but allows better lane switching decisions in noisy situations and for debugging. Not recommended for F4 boards.
4                                               Triple overlapping Notches
==========================================      =======================

.. note:: double notch option is no longer recommended since the triple notch option has been added. With a double notch, the maximum attenuation is either side of the center frequency, so on smaller aircraft with a very pronounced peak their use is usually counter productive.


.. note:: Each notch has some CPU cost so if you configure multiple notches you can end up with many notches on your aircraft. For example, triple single (no harmonics) notches, using ESC telemetry will result in 3 notches per motor or 12 total notches. For example, with F4 cpus this should be acceptable, but enabling a second group of triple notches with :ref:`INS_HNTC2_ENABLE<INS_HNTC2_ENABLE>` or multiple harmonic notches, could cause problems.



.. toctree::
    :hidden:

    Throttle Based <common-throttle-based-notch>
    RPM Sensor<common-rpm-based-notch>
    ESC Telemetry<common-esc-telem-based-notch>
    In-Flight FFT <common-imu-fft>
    Traditional Heli Notch Filter Setup<common-imu-notch-filtering-helicopter-setup>

