#include <Servo.h>
// This code should be used to control a robotic arm using inverse kinematics, 
//it takes coordinate values in the defined coordinate space before calculateing
//and actuating the end effector movement to those coordinates.
// Author: Sam Fricker
// Date: 20/12/2025

// initialise servos as variables
Servo elbow;
Servo shoulder;
Servo base;

// allocate variables calculated through experiment for each servo to move through 90 degrees
float s3_90 = 1100; //times for a 90 degree rotation
float s2_90 = 730;
float s1_90 = 1460;

// allocate initial starting positions
float phi = 0; // angle of s3 (starting position straight up)
float theta = 90; // angle of s2 (starting position straight up)
float alpha = 0; // base angle 

// command variables of angles which each servo must rotate through
float s1Angle;
float s2Angle;
float s3Angle; 


// useful constants
long double pi = 3.141592653589793238462643383279502884197;
float momentMax = 27075.6; // maximum moment calculated using CAD model(g*mm) 
float moment = ((105*16*9.81*cos(theta*pi/180))+(125+55)*cos((phi)*pi/180)*6*9.81*cos((theta - phi)*pi/180)); // initial moment

// add an increment for when the servo is moving up or down against its movement for the time delay
float upincrement = 300;
float downincrement = 60;

// introduce coordinate variables
float x = 0;
float y = 235; //mm
float z = 0;

// lengths of each arm
float l1 = 110; //mm
float l2 = 125; 


// assign the servos to each pin in arduino and start serial monitor
void setup() {
  elbow.attach(2); 
  shoulder.attach(3);
  base.attach(4);
  Serial.begin(9600);
}

// begin main operating loop
void loop() {
  // get desired coordinate values from serial
  if (Serial.available() > 0) {

    String input = Serial.readStringUntil(' ');
    // filter through coordinates broken up by commas
    int firstCommaIndex = input.indexOf(',');
    int secondCommaIndex = input.indexOf(',', firstCommaIndex + 1);

    // If both commas exist, extract x, y, and z values
    if (firstCommaIndex > 0 && secondCommaIndex > firstCommaIndex) {
      String xString = input.substring(0, firstCommaIndex);
      String yString = input.substring(firstCommaIndex + 1, secondCommaIndex); 
      String zString = input.substring(secondCommaIndex + 1);

      x = xString.toInt();
      y = yString.toInt();
      z = zString.toInt();

      Serial.print("x = ");
      Serial.println(x);
      Serial.print("y = ");
      Serial.println(y);
      Serial.print("z = ");
      Serial.println(z);
    }

    // provided the coorniates lie within a 235 mm radius circle, move end effector to desired point.
    if (sqrt(sq(x) + sq(y) + sq(z)) <= 235){
      
      // calculate target angles using inverse kinematics
      float targetalphaRAD = (atan(z/x));
      float targetphiRAD = (-acos((sq(x)+sq(y)-sq(l1)-sq(l2))/(2*l1*l2)));
      float targetThetaRAD = abs(atan(y/x)+atan((l1*sin(targetphiRAD))/(l1+l2*cos(targetphiRAD))));

      //convert to degrees
      float targetalpha = targetalphaRAD *180/pi;
      float targetphi = targetphiRAD *180/pi;
      float targetTheta = targetThetaRAD *180/pi;

      // print the desired angles to check kinematics is working
      Serial.print("targetalpha = ");
      Serial.println(targetalpha);
      Serial.print("targetphi = ");
      Serial.println(targetphi);
      Serial.print("targetTheta = ");
      Serial.println(targetTheta);

      // calculate angles to move through
      s1Angle = alpha - (targetalpha);
      s2Angle = theta - (targetTheta); // negative = anticlockwise, positive = clockwise
      s3Angle = phi - (targetphi);

      // print the angles which the servos must move through
      Serial.print("s1Angle = ");
      Serial.println(s1Angle);
      Serial.print("s2Angle = ");
      Serial.println(s2Angle);
      Serial.print("s3Angle = ");
      Serial.println(s3Angle);


      //Base movement
      if (s1Angle >= 0){
        base.write(83); // set speed of servo (speed chosen experimentally where 90 is stationary)
        delay(abs((s1Angle/90)*s1_90)); // delay for linearly interpolated ammount of time
        base.write(90); // stop movement
        alpha = (targetalpha); // set angle to target
      } else if (s1Angle < 0){
        base.write(100); // Set speed in angle direction 
        delay(abs((s1Angle/90)*s1_90)); // delay for linearly interpolated ammount of time
        base.write(90); // stop movement
        alpha = (targetalpha); // set angle to target
      }

      //Servo #2 movement
      if (s2Angle >= 0){
        // set speed in angle direction
        shoulder.write(80);
        // time delay must be linearly interpolated depending on angle with a constant added that depends on moment
        if (moment >= 0){
          delay(abs((s2Angle/90)*s2_90 - (moment/momentMax) * downincrement));
        } else if (moment < 0){
          delay(abs((s2Angle/90)*s2_90 - (moment/momentMax) * upincrement));
        }

        shoulder.write(90); // stop servo
        theta = (targetTheta); // update angle

      } else if (s2Angle < 0){
        shoulder.write(100);

        if (moment >= 0){
          delay(abs((s2Angle/90)*s2_90 + (moment/momentMax) * upincrement));
        } else if (moment < 0){
          delay(abs((s2Angle/90)*s2_90 + (moment/momentMax) * downincrement));
        }

        shoulder.write(90);
        theta = (targetTheta);
      }

      //Servo #3 Movement 
      if (s3Angle <= 0){
        elbow.write(80); // set speed in appropriate direction
        delay(abs((s3Angle/90)*s3_90)); // linear interpolation with angle for time delay
        elbow.write(90); // stop servo
        phi = (targetphi); // update angle
      } else if (s3Angle > 0){
        elbow.write(100);
        delay(abs((s3Angle/90)*s3_90));
        elbow.write(90);
        phi = (targetphi);
      }
      // recalculate moment after each movement.
      moment = ((105*16*9.81*cos(theta*pi/180))+(125+55)*cos((phi)*pi/180)*6*9.81*cos((theta - phi)*pi/180)); // range: +/-max
    }

    // print out anlges to troubleshoot
    Serial.print("phi = ");
    Serial.println(phi);
    Serial.print("theta = ");
    Serial.println(theta);
    Serial.print("alpha = ");
    Serial.println(alpha);
    Serial.print("moment (g*mm) = ");
    Serial.println(moment);
    Serial.println("------------------");
  }
}
