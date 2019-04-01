# Mongoose OS IMU Library

This library provides a simple API that describes inertial measurement units.
It implements various popular I2C and SPI IMUs behind that API. Callers of
the library can rely on the API returning consistently typed data regardless
of the choice of sensor.

# Anatomy

This library provides an API to expose various implementations of gyroscopes,
accelerometers, and magnetometers. It offers a suitable abstraction that 
presents the sensor data in a consistent way, so application authors need
not worry about the implementation details of the sensors.

Users create a `struct mgos_imu` object, to which they add sensors (gyroscope,
acellerometer and magnetometer) either via I2C or SPI. Some chips have all
three sensor types, some have only two (typically accelerometer and gyroscope)
and some have only one type. After successfully adding the sensors to the
`mgos_imu` object, reads can be performed, mostly by performing
 `mgos_imu_*_get()` calls.

All implementations offer the calls described below in the `IMU API`.
Some implementations offer chip-specific addendums, mostly setting sensitivity
and range options.

## IMU API

### IMU primitives

`struct mgos_imu *mgos_imu_create()` -- This call creates a new (opaque) object
which represents the IMU device. Upon success, a pointer to the object will be
returned. If the creation fails, NULL is returned.

`void mgos_imu_destroy()` -- This cleans up all resources associated with with
the IMU device. The caller passes a pointer to the object pointer. If any
sensors are attached (gyroscope, accelerometer, magnetometer), they will be
destroyed in turn using `mgos_imu_*_destroy()` calls. 

`bool mgos_imu_read()` -- This call schedules all sensors attached to the IMU
to be read from. It is not generally necessary to call this method directly,
as the `mgos_imu_*_get()` calls internally schedule reads from the sensors as
well.

`bool mgos_imu_accelerometer_present()` -- This returns `true` if the IMU has an
attached accelerometer sensor, or `false` otherwise.

`bool mgos_imu_gyroscope_present()` -- This returns `true` if the IMU has an
attached gyroscope sensor, or `false` otherwise.

`bool mgos_imu_magnetometer_present()` -- This returns `true` if the IMU has an
attached magnetometer, or `false` otherwise.

### IMU Sensor primitives

For each of ***accelerometer***, ***gyroscope*** and ***magnetometer***, the
following primitives exist:

`bool mgos_imu_*_create_i2c()` -- This attaches a sensor to the IMU
based on the `type` enum given, using the `i2c` bus and specified `i2caddr`
address. The function will return `true` upon success and `false` in case
either detection of the sensor, or creation of it failed.

`bool mgos_imu_*_create_spi()` -- This attaches a sensor to the IMU
based on the `type` enum given, using the `spi` bus and a specified cable select
in `cs_gpio`. The function will return `true` upon success and `false` in case
either detection of the sensor or creation of it failed.

`bool mgos_imu_*_destroy()` -- This detaches a sensor from the IMU if it exists.
It takes care of cleaning up all resources associated with the sensor, and
detaches it from the `i2c` or `spi` bus. The higher level `mgos_imu_destroy()`
call uses these lower level calls to clean up sensors as well.

`bool mgos_imu_*_get()` -- This call returns sensor data after
polling the sensor for the data. It returns `true` if the read succeeded, in
 which case the floats pointed to by `x`, `y` and `z` are filled in. If an
error occurred, `false` is returned and the contents of `x`, `y` and `z` are
unmodified. Note the units of the return values:

*   ***magnetometer*** returns units of `Gauss`.
*   ***accelerometer*** returns units of `G`.
*   ***gyroscope*** returns units of `degrees per second`.

`const char *mgos_imu_*_get_name()` -- This returns a symbolic name of the
attached sensor, which is guaranteed to be less than or equal to 10 characters
and always exist. If there is no sensor of this type attached, `VOID` will be
returned. If the sensor is not known, `UNKNOWN` will be returned. Otherwise,
the chip manufacturer / type will be returned, for example `MPU9250` or
`ADXL345` or `MAG3110`.

`bool mgos_imu_*_set_orientation()` and `bool mgos_imu_*_get_orientation()` --
Often times a 9DOF sensor will have multiple chips, whose axes do not line up
correctly. Even within a single chip the accelerometer, gyroscope and
magnetometer axes might not line up (for an example of this, see [MPU9250
chapter 9.1](http://www.invensense.com/wp-content/uploads/2015/02/PS-MPU-9250A-01-v1.1.pdf)).
To ensure that the `x`, `y`, and `z` axes on all sensors are pointed in the same
direction, we can set the orientation on the gyroscope and magnetometer.
See `mgos_imu.h` for more details and an example of how to do this.

## Supported devices

### Accelerometer

*   MPU9250 and MPU9255
*   ADXL345
*   LSM303D and LSM303DLM
*   MMA8451
*   LSM9DS1
*   LSM6DSL
*   MPU6000 and MPU6050
*   ICM20948

### Gyroscope

*   MPU9250 and MPU9255
*   L3GD20 and L3GD20H
*   ITG3205
*   LSM9DS1
*   LSM6DSL
*   MPU6000 and MPU6050
*   ICM20948

### Magnetometer

*   MAG3110
*   AK8963 (as found in MPU9250/MPU9255)
*   AK8975
*   LSM303D and LSM303DLM
*   HMC5883L
*   LSM9DS1
*   ICM20948

## Adding devices

This is a fairly straight forward process, taking the `ADXL345` accelerometer
as example:

1.  Add a new driver header and C file, say `src/mgos_imu_adxl345.[ch]`. The
    header file exposes all of the `#define`'s for registers and constants.
1.  In the header file, declare a `detect`, `create`, `destroy` and `read`
    function.
    *   `bool mgos_imu_adxl345_detect()` -- this function optionally attempts
        to detect the chip, often times by reading a register or set of
        registers which uniquely identifies it. Not all chips can actually be
        detected, so it's OK to not define this function at all.
    *   `bool mgos_imu_adxl345_create()` -- this function has to perform the
        initialization of the chip, typically by setting the right registers
        and possibly creating a driver-specific memory structure (for, say,
        coefficients or some such). If used, that memory structure is attached
        to the `user_data` pointer, and if so, the implementation of the
        `_destroy()` function must clean up and free this memory again.
    *   `bool mgos_imu_adxl345_read()` -- this function performs the chip
        specific read functionality. This will be called whenever the user asks
        for data, either by calling `mgos_imu_read()` or by calling
        `mgos_imu_*_get()`.
    *   `bool mgos_imu_adxl345_destroy()` -- this function deinitializes the
        chip, and optionally clears and frees the driver-specific memory
        structure in `user_data`. Not all chips need additional memory
        structures or deinitialization code, in which case it's OK to not
        define this function at all.
1.  Implement the `detect`, `create`, `destroy` and `read` functions in the
    source code file `src/mgos_imu_adxl345.c`.
1.  Add the device to one of the `enum mgos_imu_*_type` in `include/mgos_imu.h`.
    In the example case of adxl345 (which is an accelerometer), add it to
    `enum mgos_imu_acc_type`.
1.  Add a string version of this to function `mgos_imu_*_get_name()` so that
    callers can resolve the sensor to a human readable format. Make sure that
    the string name is not greater than 10 characters.
1.  Add the type to the `switch()` in `mgos_imu_*_create_i2c()` (or `_spi()`)
    functions.
1.  Update this document to add the driver to the list of supported drivers.
1.  Test code on a working sample, and send a PR using the guidelines laid
    out in [contributing](CONTRIBUTING.md).

It is important to note a few implementation rules when adding drivers:

*   New drivers MUST NOT change any semantics of the abstraction layer
    (`mgos_imu` and `mgos_imu_*` members) nor the `mgos_imu_*_get()` functions.
*   Implementations of drivers MUST provide bias, scaling and other
    normalization in the driver itself. What this means, in practice, is that
    the correct units must be produced (`m/s/s`, `deg/s` and `uT`).
*   Pull Requests MUST NOT mix driver and abstraction changes. Separate them.
*   Changes to the abstraction layer MUST be proven to work on all existing
    drivers, and will generally be scrutinized.

### Example driver (AK8975)

Here's an example, for the magnetometer chip `AK8975`, showing a set of commits
for each of the steps above (and honoring the driver implementation rules).

1.  Add `src/mgos_imu_ak8975.[ch]` [commit](https://github.com/mongoose-os-libs/imu/commit/64d29e32f7633ec22c5296c27c3faf6df75f929d)
1.  Extend `enum mgos_imu_mag_type` in `include/mgos_imu.h` [commit](https://github.com/mongoose-os-libs/imu/commit/67b121c9f0dd511e3f3b5279c310c2e135895d02)
1.  Add to `mgos_imu_magnetometer_get_name()` in `src/mgos_imu_magnetometer.c` [commit](https://github.com/mongoose-os-libs/imu/commit/286d53a199df124e793e9e428657fcd28ea7b3c3)
1.  Add to `mgos_imu_magnetometer_create_i2c()` in `src/mgos_imu_magnetometer.c` [commit](https://github.com/mongoose-os-libs/imu/commit/8e33a0fbd805b5f840761266c03163deeb1cc3f3)

# Example Code

An example program that creates an IMU of type `MPU9250` (which has an
accelerometer, a gyroscope and a magnetometer all in one tiny package):

```
#include "mgos.h"
#include "mgos_i2c.h"
#include "mgos_imu.h"

static void imu_cb(void *user_data) {
  struct mgos_imu *imu = (struct mgos_imu *)user_data;
  float ax, ay, az;
  float gx, gy, gz;
  float mx, my, mz;

  if (!imu) return;

  if (mgos_imu_accelerometer_get(imu, &ax, &ay, &az))
    LOG(LL_INFO, ("type=%-10s Accel X=%.2f Y=%.2f Z=%.2f",
                  mgos_imu_accelerometer_get_name(imu), ax, ay, az));
  if (mgos_imu_gyroscope_get(imu, &gx, &gy, &gz))
    LOG(LL_INFO, ("type=%-10s Gyro  X=%.2f Y=%.2f Z=%.2f",
                  mgos_imu_gyroscope_get_name(imu), gx, gy, gz));
  if (mgos_imu_magnetometer_get(imu, &mx, &my, &mz))
    LOG(LL_INFO, ("type=%-10s Mag   X=%.2f Y=%.2f Z=%.2f",
                  mgos_imu_magnetometer_get_name(imu), mx, my, mz));
}

enum mgos_app_init_result mgos_app_init(void) {
  struct mgos_i2c *i2c = mgos_i2c_get_global();
  struct mgos_imu *imu = mgos_imu_create();
  struct mgos_imu_acc_opts acc_opts;
  struct mgos_imu_gyro_opts gyro_opts;
  struct mgos_imu_mag_opts mag_opts;

  if (!i2c) {
    LOG(LL_ERROR, ("I2C bus missing, set i2c.enable=true in mos.yml"));
    return false;
  }

  if (!imu) {
    LOG(LL_ERROR, ("Cannot create IMU"));
    return false;
  }

  acc_opts.type = ACC_MPU9250;
  acc_opts.scale = 16.0; // G
  acc_opts.odr = 100;    // Hz
  if (!mgos_imu_accelerometer_create_i2c(imu, i2c, 0x68, &acc_opts))
    LOG(LL_ERROR, ("Cannot create accelerometer on IMU"));

  gyro_opts.type = GYRO_MPU9250;
  gyro_opts.scale = 2000; // deg/sec
  gyro_opts.odr = 100;    // Hz
  if (!mgos_imu_gyroscope_create_i2c(imu, i2c, 0x68, &gyro_opts))
    LOG(LL_ERROR, ("Cannot create gyroscope on IMU"));

  mag_opts.type = MAG_AK8963;
  mag_opts.scale = 12.0; // Gauss
  mag_opts.odr = 10;     // Hz
  if (!mgos_imu_magnetometer_create_i2c(imu, i2c, 0x0C, &mag_opts))
    LOG(LL_ERROR, ("Cannot create magnetometer on IMU"));

  mgos_set_timer(1000, true, imu_cb, imu);
  return true;
}
```

# Demo Code

For a cool demo, take a look at my [Demo Apps](https://github.com/mongoose-os-apps/imu-demo):

![Chrome Demo](docs/chrome-client.png)

# Disclaimer

This project is not an official Google project. It is not supported by Google
and Google specifically disclaims all warranties as to its quality,
merchantability, or fitness for a particular purpose.
