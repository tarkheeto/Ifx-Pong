# IFX-Man
The goal of the project is to write a game where the user can play by vertically moving a controller Up and Down. This vertical motion has to be captured and its respective data sent wirelessly to the game station, where a python game is to be played


<div style="text-align:center">
    <img src="Photos/Game Start Screen.PNG" alt="Current Setup" width="720"/>
</div>

## Gameplay
+	Initially, the game will calibrate. A warning may be issued to keep the controller stationary.
+	Once calibrated, the game can be started with the 's' key.
+	The character moves up or down based on the motion detected from the controller.
+	Clouds spawn from the right and move towards the character.
+	If the character collides with a cloud, the game ends.
+	The objective is to avoid the clouds and get a high score.
+	The higher the score the faster the clouds




## Current Setup
2 XMC2Go each connected to a DPS310 sensor. Both uControllers are connected to a computer and both are simultaneously serially transmiting their air pressure readings.

<div style="text-align:center">
    <img src="Photos/setupExplanation.PNG" alt="Current Setup" width="480"/>
</div>

These two microcotrollers are setup as follows:
   * station (placed on a reference surface)
   * Dynamic Controller (Held in the hand of the user)

The computer then calculates the difference delta between the pressure measured at the station and the dynamic controller. This delta value is used as an offset, where an upper and lower threshold is calculated for detecting the vertical motion of the dynamic controller



This setup allows maximum noise negation as only the difference delta between both readings of the sensors is taken into account


**ISSUE:**: Overtime the **delta** value drifts


### Serial Comunication between microcontrollers and Computer
For serial communication we utilize the '**serial**' python module. 
```python
import serial
```
**Initialization of Serial Ports:**
```python
base_station = serial.Serial('COM30', 9600, timeout=1)
dynamic_controller = serial.Serial('COM36', 9600, timeout=1)
```
We already established in advance that the COM ports for the base station and dynamic controller are respectively **COM30** and **COM36**. So all we had to do was to write these two lines in order to inform our code. 
```python
def get_float_from_port(port):
	"""Try reading a line from a port and convert to float."""
	try:
		line = port.readline().decode().strip()
		return float(line)
	except ValueError:
		print(f"Couldn't convert '{line}' to float.")
		return None
```
* **port.readline()**: This reads a line from the serial port until a newline character is found. It returns the line as bytes.

* **decode()**: This decodes the bytes into a string using a default UTF-8 encoding.
* **strip()**: This removes any leading or trailing white spaces or newline characters from the string.

**Note**: Do not directly add the get_float_from_port() function into your game loop because then your code flow will be limited by the times at which you receive serial messages from the microcontrollers. Instead execute the function in an if condition that checks if there is data available on the serial bus as follows:

```python
	if (base_station.in_waiting > 0) and (dynamic_controller.in_waiting > 0) :
		base_value = get_float_from_port(base_station)
		controller_value = get_float_from_port(dynamic_controller)
```

### calibration
for the game to run and the characters to appear the **firstCalibrationFlag** has to be set to **True**. This ensures that calibration is performed at least once before gameplay starts.
``` python
		if (counterFirstCalibration<10):
		  counterFirstCalibration+=1
			bufFirstCalibration.add(int(controller_value - base_value))
			if(counterFirstCalibration==10):
				calibratedOffset=bufFirstCalibration.average()
				upperThreshold = calibratedOffset + 2
				lowerThreshold = calibratedOffset -2
				firstCalibrationFlag=True
```
This code snippet runs right after the booting up the game it essentially captures 20 measurements (10 from each microcontroller) performs the delta calculation and takes the average of them. The user also has the option to recalibrate by pressing the keyboard buton '**a**' ; this resets the counterFirstCalibration variable to 0. 

# Characters

## The ifxMan Class
This class represents the primary character (or sprite) that the player controls.

+	**Initialization Method (__init__)**

+	**super().__init__()**: This calls the initialization method of the parent class (pygame.sprite.Sprite), ensuring that all the built-in properties and methods of the Sprite class are inherited.

+	**self.image**: Loads the image of the character using **pygame.image.load()**. The convert_alpha() method ensures that the alpha channel (transparency) of the image is preserved.

+	**self.image=pygame.transform.scale_by(self.image, 0.1)**: This scales the image by 10%, making the sprite smaller.

+	**self.rect**: Retrieves a rectangular area from the image which will be used for positioning, collision detection, etc. It's set to appear in the middle top portion of the window.

+	**self.mask**: Creates a mask for the image. Masks are used in Pygame for pixel-perfect collision detection.

+	**collisionsWithClouds Method**: This method checks if the ifxMan sprite (or character) collides with any sprite in the cloudGroup.

+	**pygame.sprite.spritecollide()**: This function checks for collisions between a sprite and a group of sprites. The method returns a list of all Sprites in a Group that intersect with another Sprite. The pygame.sprite.collide_mask argument means that the method uses the mask for collision detection.
If a collision is detected, it returns True.


## The cloudClass
This class represents the clouds (or obstacles) that spawn and move towards the ifxMan.

+	**Initialization Method (__init__):**
Like with ifxMan, this class also inherits from pygame.sprite.Sprite.
self.image: Loads the cloud image.
self.image=pygame.transform.scale_by(self.image,0.8): Scales the cloud image by 80%.

+	**self.rect:** Gets the rectangular area of the cloud image and sets its position based on the pos argument provided when creating a cloud instance.
self.mask: Generates a mask for the cloud, for pixel-perfect collision detection.

+	**self.pos:** Initializes the position of the cloud based on the rectangle's center. This is important for smooth animations.
self.direction: Sets the movement direction of the cloud. In this case, it's set to move leftwards (-1, 0).

+	**self.speed:** Sets the movement speed of the cloud.
update Method

+	**The update method** is called every game loop to update the cloud's position.
The position of the cloud (self.pos) is updated based on its direction and speed. The dt factor ensures that movement is consistent regardless of how fast the loop runs.
If the cloud moves out of the screen (its right edge is less than 0), it gets destroyed (or 'killed') to save memory and processing.


## Collision Detection

The actual collision detection is performed in the collisionsWithClouds method of the ifxMan class:

```python
def collisionsWithClouds(self):
    if pygame.sprite.spritecollide(self, cloudGroup, True, pygame.sprite.collide_mask):
        return True
```

+	**pygame.sprite.spritecollide(...)**: This function checks for collisions between a sprite (the ifxMan in this case) and a group of sprites (the cloudGroup).

+	The **True** argument indicates that when a collision is detected, the collided sprite in the group should be removed (or 'killed'). This means a cloud will disappear upon colliding with the ifxMan.

+	**pygame.sprite.collide_mask**: This is a collision detection function that checks for collisions using masks (pixel-perfect collision detection). So, instead of just checking if the rectangles of the sprites overlap (which can be imprecise), it checks if the actual pixels (or filled parts) of the sprites overlap, ensuring a much more accurate collision detection.

When a collision is detected, the **collisionsWithClouds** method returns **True**, indicating that the **ifxMan** character has collided with a cloud.


## Spawning the Clouds
The spawning mechanism for the clouds is event-driven, which means it's based on Pygame events. Specifically, there's a timer event set up to trigger the cloud spawning.

### Custom Timer Event:
Firstly, a custom event type for the cloud spawning timer is created:
```python
cloudTimer = pygame.event.custom_type()
```
## Setting up the Timer:
Then, the timer is set to trigger the cloudTimer event every 5000 milliseconds (or 5 seconds):

```python
pygame.time.set_timer(cloudTimer, 5000)
```
This means that every 5 seconds, a **cloudTimer** event will be added to the Pygame event queue.

## Handling the Timer Event:
Inside the main game loop, events are continuously polled and checked. Whenever the cloudTimer event occurs, a new cloud is spawned:
```python
for event in pygame.event.get():
    if gameMode:
        if event.type == cloudTimer:
            cloudClass((2200, randint(0, 1080)), score * 50 + 50, cloudGroup)
            print("schblanga")
```
When the **cloudTimer** event is detected, the cloudClass is instantiated, effectively creating a new cloud sprite:

+	**(2200, randint(0, 1080))** sets the initial position of the cloud. The X-coordinate is **2200** (presumably off-screen to the right, so it moves into the view), and the Y-coordinate is randomly generated between **0** and **1080** to give vertical variety to the spawning.

+	**score * 50 + 50** is the speed parameter. As the score increases, clouds move faster, making the game progressively harder.

+	**cloudGroup** is the sprite group to which the cloud sprite is added. Storing sprites in groups allows for easier batch processing of sprites, for operations such as drawing, updating, or collision checking


## Off-screen Killing:
Inside the **cloudClass** definition, there's an **update** method which ensures that if a cloud goes entirely off the screen to the left (i.e., its right edge **(self.rect.right)** is less than 0), it gets removed (or 'killed'):
```python
if self.rect.right < 0:
    self.kill()
```
This is a good optimization. By removing sprites that are no longer visible or needed, you reduce the computational work and memory usage.









  
