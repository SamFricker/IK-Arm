// RealTimeControl is code for an arduino that should be used to control a robotic arm with the keyboard of the powering computer 
#include <Servo.h> // include servo library to control servo motors

// assign servos names of joint positions
Servo elbow;    // servo controlling elbow joint (s3)
Servo shoulder; // servo controlling shoulder joint (s2)
Servo base;     // servo controlling base rotation (s1)

// experimentally determined times for a 90 degree rotation at speeds
float s3_90 = 1100; //delay (ms) for elbow to rotate 90 degrees
float s2_90 = 730;  //delay (ms) for shoulder to rotate 90 degrees
float s1_90 = 1460; //delay (ms) for base to rotate 90 degrees

// initialise starting position angles
float phi = 90;   // angle of s3 (starting position straight up)
float theta = 90; // angle of s2 (starting position straight up)
float alpha = 0;  // base angle (starting facing forward)

// command variables of angles which each servo must rotate through
float s1Angle; // commanded rotation for base
float s2Angle; // commanded rotation for shoulder
float s3Angle; // commanded rotation for elbow

//constants
long double pi = 3.141592653589793238462643383279502884197; // high precision value of pi for calculations
float momentMax = 27075.6; //calculated maximum value of moment (g*mm)
float moment = ((105*16*9.81*cos(theta*pi/180))+(125+55)*cos((90-phi)*pi/180)*6*9.81*cos((theta + phi - 90)*pi/180)); // initial moment based on starting angles

// experimentally determined time increments to counteract moments
float upincrement = 300;   // additional delay when lifting against gravity
float downincrement = 30;  // reduced delay when moving with gravity

// assign servos to each pin and start serial monitor
void setup() {
  elbow.attach(2);    // attach elbow servo to pin 2
  shoulder.attach(3); // attach shoulder servo to pin 3
  base.attach(4);     // attach base servo to pin 4
  Serial.begin(9600); // begin serial communication at 9600 baud
}

// start main operating loop
void loop() {

  // get serial input as a number between 1 and 9
  if (Serial.available() > 0) { // check if data is available from serial
    String input = Serial.readStringUntil('\n'); // read full input until newline
    char v = input[0]; // first character determines command type
    String angleStr = input.substring(1); // remaining string represents angle value

    //S3

    if (v == '5') {
      elbow.write(90);    // reset elbow to neutral position
      shoulder.write(90); // reset shoulder to neutral position
      base.write(90);     // reset base to neutral position
    } else if (v == '4'){
      s3Angle = angleStr.toFloat(); // convert input angle to float
      elbow.write(80); // move elbow in one direction
      delay((s3Angle/90)*s3_90); // scale delay proportionally to desired angle
      elbow.write(90); // stop movement
      phi = (phi + s3Angle); // update stored elbow angle
    } else if (v == '6'){
      s3Angle = angleStr.toFloat(); // convert input angle to float
      elbow.write(100); // move elbow in opposite direction
      delay((s3Angle/90)*s3_90); // scale delay proportionally to desired angle
      elbow.write(90); // stop movement
      phi = (phi - s3Angle); // update stored elbow angle
    }

    //S2
    
    if (v == 'y'){
      s2Angle = angleStr.toFloat(); // convert input angle to float
      shoulder.write(80); // move shoulder upward direction
      if (moment >= 0){
        delay((s2Angle/90)*s2_90 - (moment/momentMax) * downincrement); // adjust delay based on load assisting motion
      } else if (moment < 0){
        delay((s2Angle/90)*s2_90 - (moment/momentMax) * upincrement); // adjust delay when working against load
      }
      shoulder.write(90); // stop movement
      theta = theta - s2Angle; // update stored shoulder angle
    } else if (v == 'r'){
      s2Angle = angleStr.toFloat(); // convert input angle to float
      shoulder.write(100); // move shoulder downward direction
      if (moment >= 0){
        delay((s2Angle/90)*s2_90 + (moment/momentMax) * upincrement); // adjust delay when lifting load
      } else if (moment < 0){
        delay((s2Angle/90)*s2_90 + (moment/momentMax) * downincrement); // adjust delay when load assists motion
      }
      shoulder.write(90); // stop movement
      theta = theta + s2Angle; // update stored shoulder angle
    }

    //Base

    if (v == 'd'){
      s1Angle = angleStr.toFloat(); // convert input angle to float
      base.write(83); // rotate base in one direction
      delay((s1Angle/90)*s1_90); // scale delay based on angle
      base.write(90); // stop movement
      alpha = (alpha - s1Angle); // update stored base angle
    } else if (v == 'g'){
      s1Angle = angleStr.toFloat(); // convert input angle to float
      base.write(100); // rotate base in opposite direction
      delay((s1Angle/90)*s1_90); // scale delay based on angle
      base.write(90); // stop movement
      alpha = (alpha + s1Angle); // update stored base angle
    }

    moment = ((105*16*9.81*cos(theta*pi/180))+(125+55)*cos((90-phi)*pi/180)*6*9.81*cos((theta + phi - 90)*pi/180)); // recalculate moment after movement (range: +/-max)

    Serial.println("------------------"); // separator for readability in serial monitor
    Serial.print("phi = "); // label for elbow angle
    Serial.println(phi); // print elbow angle
    Serial.print("theta = "); // label for shoulder angle
    Serial.println(theta); // print shoulder angle
    Serial.print("alpha = "); // label for base angle
    Serial.println(alpha); // print base angle
    Serial.print("moment (g*mm) = "); // label for calculated moment
    Serial.println(moment); // print current moment value
  }

}