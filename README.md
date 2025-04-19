#region VEXcode Generated Robot Configuration
from vex import *
import urandom

# Brain should be defined by default
brain = Brain()

# Robot configuration code
left_motor_a = Motor(Ports.PORT21, GearSetting.RATIO_6_1, True)
left_motor_b = Motor(Ports.PORT2, GearSetting.RATIO_6_1, True)
left_motor_c = Motor(Ports.PORT19, GearSetting.RATIO_6_1, True)
left_drive_smart = MotorGroup(left_motor_a, left_motor_b, left_motor_c)
right_motor_a = Motor(Ports.PORT18, GearSetting.RATIO_6_1, False)
right_motor_b = Motor(Ports.PORT17, GearSetting.RATIO_6_1, False)
right_motor_c = Motor(Ports.PORT4, GearSetting.RATIO_6_1, False)
right_drive_smart = MotorGroup(right_motor_a, right_motor_b, right_motor_c)
drivetrain = DriveTrain(left_drive_smart, right_drive_smart, 319.19, 295, 40, MM, 1)
IntakeBelt = Motor(Ports.PORT5, GearSetting.RATIO_6_1, False)
IntakeWheels = Motor(Ports.PORT16, GearSetting.RATIO_18_1, True)
LadyBrown = Motor(Ports.PORT20, GearSetting.RATIO_18_1, False)
ColorSort = Optical(Ports.PORT12)
mogoPiston1 = DigitalOut(brain.three_wire_port.g)
mogoPiston2 = DigitalOut(brain.three_wire_port.h)
leftClear = DigitalOut(brain.three_wire_port.e)
rightClear = DigitalOut(brain.three_wire_port.f)
controller_1 = Controller(PRIMARY)
IntakeBelt.set_velocity(100, PERCENT)
IntakeWheels.set_velocity(100, PERCENT)
LadyBrown.set_velocity(100, PERCENT)

# wait for rotation sensor to fully initialize
wait(30, MSEC)

# Make random actually random
def initializeRandomSeed():
    wait(100, MSEC)
    random = brain.battery.voltage(MV) + brain.battery.current(CurrentUnits.AMP) * 100 + brain.timer.system_high_res()
    urandom.seed(int(random))

# Set random seed 
initializeRandomSeed()

def play_vexcode_sound(sound_name):
    print("VEXPlaySound:" + sound_name)
    wait(5, MSEC)

wait(200, MSEC)
print("\033[2J")

# define variables used for controlling motors based on controller inputs
controller_1_right_shoulder_control_motors_stopped = True
drivetrain_l_needs_to_be_stopped_controller_1 = False
drivetrain_r_needs_to_be_stopped_controller_1 = False
toggle_mogo = False
toggle_mogo_pressed = False
toggle_left_clear = False
toggle_left_clear_pressed = False
toggle_right_clear = False
toggle_right_clear_pressed = False

# FSM for LadyBrown Motor
# Define initial state and position variables
state = 0
positions = [0, 125, 500]  # Updated position 135 to 125

# Define a function to handle state transitions
def update_lady_brown_position():
    global state
    if state == 0:
        LadyBrown.spin_to_position(positions[0], DEGREES, wait=True)
    elif state == 1:
        LadyBrown.spin_to_position(positions[1], DEGREES, wait=True)
    elif state == 2:
        LadyBrown.spin_to_position(positions[2], DEGREES, wait=True)

# Define a task that will handle state transitions based on button press
def fsm_task():
    global state
    while True:
        # If button L2 is pressed, move to the next state (go forward in positions)
        if controller_1.buttonL2.pressing():
            state = (state + 1) % 3  # Cycle through 0, 1, 2
            update_lady_brown_position()  # Update the motor position based on the new state

        # If button L1 is pressed, move to the previous state (go backward in positions)
        if controller_1.buttonL1.pressing():
            state = (state - 1) % 3  # Cycle through 0, 1, 2 in reverse
            update_lady_brown_position()  # Update the motor position based on the new state

        # Wait before checking again
        wait(20, MSEC)

# Start the FSM task in a separate thread
fsm_thread = Thread(fsm_task)

# define a task that will handle monitoring inputs from controller_1
def rc_auto_loop_function_controller_1():
    global drivetrain_l_needs_to_be_stopped_controller_1, drivetrain_r_needs_to_be_stopped_controller_1, controller_1_right_shoulder_control_motors_stopped, remote_control_code_enabled
    global toggle_mogo, toggle_mogo_pressed, toggle_left_clear, toggle_left_clear_pressed, toggle_right_clear, toggle_right_clear_pressed

    ColorSort.set_light_power(100, PERCENT)
    ColorSort.set_light(Color.WHITE)

    while True:
        if remote_control_code_enabled:
            drivetrain_left_side_speed = controller_1.axis2.position() + controller_1.axis4.position()
            drivetrain_right_side_speed = controller_1.axis2.position() - controller_1.axis4.position()

            if drivetrain_left_side_speed < 5 and drivetrain_left_side_speed > -5:
                if drivetrain_l_needs_to_be_stopped_controller_1:
                    left_drive_smart.stop()
                    drivetrain_l_needs_to_be_stopped_controller_1 = False
            else:
                drivetrain_l_needs_to_be_stopped_controller_1 = True

            if drivetrain_right_side_speed < 5 and drivetrain_right_side_speed > -5:
                if drivetrain_r_needs_to_be_stopped_controller_1:
                    right_drive_smart.stop()
                    drivetrain_r_needs_to_be_stopped_controller_1 = False
            else:
                drivetrain_r_needs_to_be_stopped_controller_1 = True

            if drivetrain_l_needs_to_be_stopped_controller_1:
                left_drive_smart.set_velocity(drivetrain_left_side_speed, PERCENT)
                left_drive_smart.spin(FORWARD)

            if drivetrain_r_needs_to_be_stopped_controller_1:
                right_drive_smart.set_velocity(drivetrain_right_side_speed, PERCENT)
                right_drive_smart.spin(FORWARD)

            # Color sensor sorting logic
            if ColorSort.is_near_object():
                if ColorSort.color() == Color.RED:
                    IntakeWheels.spin(FORWARD)
                    IntakeBelt.spin(FORWARD)
                elif ColorSort.color() == Color.BLUE:
                    IntakeWheels.stop()
                    IntakeBelt.stop()
                    wait(300, MSEC)
                    IntakeWheels.spin(REVERSE)
                    IntakeBelt.spin(REVERSE)
            else:
                IntakeWheels.spin(FORWARD)
                IntakeBelt.spin(FORWARD)

            if controller_1.buttonR1.pressing():
                IntakeWheels.spin(FORWARD)
                IntakeBelt.spin(FORWARD)
                controller_1_right_shoulder_control_motors_stopped = False
            elif controller_1.buttonR2.pressing():
                IntakeWheels.spin(REVERSE)
                IntakeBelt.spin(REVERSE)
                controller_1_right_shoulder_control_motors_stopped = False
            elif not controller_1_right_shoulder_control_motors_stopped:
                IntakeWheels.stop()
                IntakeBelt.stop()

            # Mogo Piston Toggle (Prevents Rapid Switching)
            if controller_1.buttonDown.pressing():
                if not toggle_mogo_pressed:
                    toggle_mogo = not toggle_mogo
                    mogoPiston1.set(toggle_mogo)
                    mogoPiston2.set(toggle_mogo)
                    toggle_mogo_pressed = True
            else:
                toggle_mogo_pressed = False

            # Left Clear Piston Toggle
            if controller_1.buttonLeft.pressing():
                if not toggle_left_clear_pressed:
                    toggle_left_clear = not toggle_left_clear
                    leftClear.set(toggle_left_clear)
                    toggle_left_clear_pressed = True
            else:
                toggle_left_clear_pressed = False

            # Right Clear Piston Toggle
            if controller_1.buttonRight.pressing():
                if not toggle_right_clear_pressed:
                    toggle_right_clear = not toggle_right_clear
                    rightClear.set(toggle_right_clear)
                    toggle_right_clear_pressed = True
            else:
                toggle_right_clear_pressed = False

        wait(20, MSEC)

# define variable for remote controller enable/disable
remote_control_code_enabled = True

rc_auto_loop_thread_controller_1 = Thread(rc_auto_loop_function_controller_1)

#endregion VEXcode Generated Robot Configuration

# ------------------------------------------
# 
# 	Project:      VEXcode Project
# 	Author:       VEX
# 	Created:
# 	Description:  VEXcode V5 Python Project
# 
# ------------------------------------------

# Begin project code
