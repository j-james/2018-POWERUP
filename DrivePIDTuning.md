## Drive PID Tuning

Drivetrain PID tuning differs from simpler PID scenarios due to the
number of different components in the drivetrain.  We'd like to control
the drivetrain with the fewest number of knobs, but each drive assembly
(motor, gearbox, chain, belt, axle) has a unique friction coefficient.
Thus, a unit of power delivered to the left drivetrain may produce very
different results than the same unit of power delivered to the right
drivetrain.  Add to this the fact that our PathFollowing implementation
represents a higher-level control system and it has a number of control
knobs that must be tuned in addition to the parameters for the low-level
motor controller.

### Our basic tuning strategy from 3/9/18:

1. make all gains except Kp = 0.  Make Kp small and verify that for a
straight line path, it takes a while to reach the target velocity. The
dashboard speed and speed-error graph is very helpful when tuning the
velocity control parameters.

2. increase Kp to try to get up to speed faster.  When you start seeing
oscillations in the error, you've gone a bit too far.

3. if the robot doesn't drive straight, this is likely because a unit of
control (ie PctOutput) means a different thing for each side of the
drivetrain.  We can nudge the Kf parameter slightly for the sluggish side.  
Note that Kf is a generally a tricky parameter, since its ideal value is
proportional to the target speed (which may change over the course of the
path and we have not delved into varying the Kf value over time).

4. Only once Kp is figured out is it worth exploring the value of Ki and/or
Kd. In theory, Ki should help to get to target faster, but too-large values
produce oscillations.   In theory, Kd should help to reduce oscillations
but may be very sensitive to noise in the error measurement (which can come
about due to irregular sampling intervals among other things).

### Some notes on tuning gains for TalonSRX

* In the TalonSRX firmware, motor control is represented in integer
 units between -1023 and 1023. We can thing of these as PctOutput
 values between -1 and 1.  Note that motor output depends upon battery
 voltage. As the battery voltage decreases (over a match), one unit
 of power-control is expected to produce less force.  If this becomes a
 problem, we may need to investigate the SRX's voltage compensation
 features. Please note: **gain values for SRX should always be integers**.
 Small floating point percentage values are equivalent to 0 and will
 have no effect.

* Velocity setpoints are represented on the TalonSRX in native encoder
 units per 100ms.  Our software converts our standard velocity unit
 (inches/sec) into SRX velocity units behind the scenes.

* **Kp**:  is the proportional gain. It modifies the closed-loop output
 by a proportion (the gain value) of the closed-loop error. Kp is specified
 in output unit per error unit. For example, a value of 102 is ~9.97%
 (which is 102/1023) output per 1 unit of Closed-Loop Error. Remember that
 the output units differ between position and velocity modes, so it's
 expected that Kp values will differ substantially for each control mode.
 Tune this until the sensed value is close to the target under typical load.
 Many prefer to simply double the P-gain until oscillations occur, then
 reduce accordingly.

* **Ki**: is the integral gain. It modifies the closed-loop output according
 to the integral error (summation of the closed-loop error each iteration).
 I gain is specified in output units per integrated error. For example, a
 value of 10 equates to ~0.97% for each accumulated error. Integral
 accumulation is done every 1ms. In our experience, Ki has been impossible
 to use - even small units of control result in wild and wacky behavior.
 To address this, we need to explore controlling additional integration
 parameters:

 * ```configMaxIntegralAccumulator()``` : establishes a maximum error value
   measured in closed loop error units X 1ms. Aka, izone.
 * ```setIntegralAccumulator()```:  typically used to clear/zero the integral
  accumulator, however some use cases may require seeding the accumulator
  for a faster response. In addition manual clear, there are certain cases
  where the integral accumulator is automatically cleared for more predicable
  motor response.
    * Whenever the control mode of a Talon is changed.
    * When a Talon is in the disabled state.
    * When the motor control profile slot has changed.
    * When the Closed Loop Error’s magnitude is smaller than the
      “Allowable Closed Loop Error” (ie we reach the target).

* **Kd**: is the derivative gain. It modifies the closed-loop output
 according to the derivative error (change in closed-loop error each
 iteration). Kd is specified in output units per derivative error.
 For example a value of 102 equates to ~9.97% (which is 102/1023)
 per change of Sensor Position/Velocity unit per 1ms.

* **Kf**: is the feed-forward gain. Feed-Forward is typically used in
 velocity and motion profile/magic closed-loop modes. Kf is multiplied
 directly by the set point passed into the programming API. The result
 of this multiplication is in motor output units [-1023, 1023]. This
 allows the robot to feed-forward using the target set-point. In order
 to calculate feed-forward, you will need to measure your motor's
 velocity at a specified percent output (preferably an output close
 to the intended operating range). You can see the measured velocity
 in a few different ways. The fastest is to usually do a self-test in
 the web-based interface. This will give you both your velocity and
 your percent output. Kf is then calculated using the following formula:

    ```Kf = ([Percent Output] x 1023) / [Velocity]```

 This suggests that Kf, different from other controls, is represented
 as a floating point number between -1.0 and 1.0 and conveys the power
 requirements to attain the target velocity under normal load. Interestingly,
 this formulation suggests that the Kf is usable at a variety of set points.
 In our PathFollowing, implementation, we regularly change velocity along
 as we follow the path.  _Only if we've proved that Kf is proportional to
 velocity for our drivetrain, should we rely heavily on Kf._

* **Nominal Output**: The _minimal_ or _weakest_ motor output allowed if
 the output is nonzero. This is expressed using two signals:
 “Positive Nominal Output” and “Negative Nominal Output”, to uniquely
 describe a limit for each direction. If the motor-output is too _weak_,
 the robot application can use these signals to promote the motor-output
 to a minimum limit. With this the robot application can ensure the
 motor-output is large enough to drive the mechanism. Typically in a
 closed-loop system, this is accomplished with Integral gain, however
 this method may be a simpler alternative as there is no risk of
 Integral wind-up.

## References

* https://github.com/CrossTheRoadElec/Phoenix-Documentation#closed-loop-ramping
* https://www.crossco.com/blog/basics-tuning-pid-loops

### If you prefer to read code
 Note that this code doesn't capture the API conventions for setting
 java-side parameter values down to the motor controller.  Specifically,
 questions of where the fixed-point conversions take place aren't captured
 here.
```
/**
 * 1ms process for PIDF closed-loop.
 * @param pid ptr to pid object
 * @param pos signed integral position (or velocity when in velocity mode).
 * The target pos/velocity is ramped into the target member from caller's 'in'.
 * PIDF is traditional, unsigned coefficients for P,i,D, signed for F.
 * Target pos/velocity is feed forward.
 *
 * Izone gives the abilty to autoclear the integral sum if error is wound up.
 * @param oneDirOnly when using positive only sensor, keep the closed-loop
 * from outputting negative throttle.
 *
 * Note: The PID_Mux_Unsigned and PID_Mux_Signed routines are merely
 * fixed-point multiply functions.
*/
void PID_Calc1Ms(pid_t * pid, MotorControlProfile_t * slot,
    int32_t pos, int8_t oneDirOnly, int16_t peakOutput, int periodMs)
{
    if (++pid->timeMs < periodMs) {
        /* don't run yet, keep out the same */
        return;
    }

    /* time to fire, reset time */
    pid->timeMs = 0;
    /* Save last sensor measurement for debugging. */
    pid->lastInput = pos;
    /* calc error : err = target - pos*/
    int32_t err = pid->target - pos;
    pid->err = err;
    /*abs error */
    int32_t absErr = err;
    if(err < 0)
        absErr = -absErr;
    /* integrate error and handle the first pass since PID_Reset() */
    if(0 == pid->notFirst){
        /* first pass since reset/init */
        pid->iAccum = 0;
        /* also tare the before ramp throt */
        pid->out = BXDC_GetThrot(); /* the save the current ramp */
    }
    else if((!slot->IZone) || (absErr < slot->IZone) ){
        /* izone is not used OR absErr is within iZone */
        pid->iAccum += err;
    }
    else{
        pid->iAccum = 0;
    }
    /* max iaccum */
    if (slot->maxIAccum == 0) {
        /* don't cap */
    }
    else if (pid->iAccum > slot->maxIAccum) {
        pid->iAccum = slot->maxIAccum;
    }
    else if (pid->iAccum < -slot->maxIAccum) {
        pid->iAccum = -slot->maxIAccum;
    }
    else {
        /* leave it alone */
    }
    /* dErr/dt */
    if(pid->notFirst){
        /* calc dErr */
        pid->dErr = pid->prevPos - pos
    }
    else{
        /* clear dErr */
        pid->dErr = 0;
    }
    /* if error is within the allowable range, we will clear it and clear the integral accum */
    uint16_t allowClosedLoopEr = slot->allowableErr;
    if(absErr < allowClosedLoopEr){
        /* clear Iaccum */
        pid->iAccum = 0;
        /* clear dErr */
        pid->dErr = 0;
        /* wipe this out so PID response is identical to no error */
        err = 0;
    }

    //------------------ P ------------------------//
    pid->out = PID_Mux_Unsigned(err, slot->P);

    //------------------ I ------------------------//
    if(pid->iAccum && slot->I){
        /* our accumulated error times I gain. If you want the robot
         * to creep up then pass a nonzero Igain
         */
        pid->out += PID_Mux_Unsigned(pid->iAccum, slot->I);
    }

    //------------------ D ------------------------//
    /* derivative gain, if you want to react to sharp changes in error
      (smooth things out).
      */
    pid->out += PID_Mux_Unsigned(pid->dErr, slot->D);

    //------------------ F ------------------------//
    /* feedforward on the set point */
    pid->out += PID_Mux_Signed(pid->target, slot->F);

    /* arm for next pass */
    pid->prevPos = pos; /* save the prev pos for D */
    pid->notFirst = 1; /* already serviced first pass */

    if(oneDirOnly > 0){
        /* positive only */
        if(pid->out < 0)
            pid->out = 0;
    }
    else if(oneDirOnly < 0){
        /* negative only */
        if(pid->out > 0)
            pid->out = 0;
    }
    else{
        /* no bounds */
    }
    /* output cap */
    if (pid->out > +peakOutput) {
        pid->out = +peakOutput;
    }
    else if (pid->out < -peakOutput) {
        pid->out = -peakOutput;
    }
}
```