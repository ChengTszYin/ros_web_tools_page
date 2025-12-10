<script setup>
import {ref, onMounted, onUnmounted} from 'vue'
import '../public/mjpegcanvas.min.js';

const ros = ref(null)
const rosbridge_address = ref('')
const connected = ref(false)
const dragCircleStyle = ref({
  left: '50%',
  top: '50%'
})
const isDragging = ref(false)
const dragZoneRect = ref(null) 
const position = ref({x:0, y:0, z:0})

const connect_robot = () => {   //ws://192.168.195.17:9090 // ws://192.168.0.39:9090
  if(!rosbridge_address.value)    
  {
    alert("ip cannot empty")
    return
  }
    ros.value = new ROSLIB.Ros({
    url: rosbridge_address.value
  })

  ros.value.on('connection', ()=> {
    console.log('connect')
    connected.value = true
    let topic = new ROSLIB.Topic({
      ros: ros.value,
      name: 'odom',
      messageType: 'nav_msgs/Odometry'
    })
    topic.subscribe((message) => {
      position.value = {
        x: message.pose.pose.position.x,
        y: message.pose.pose.position.y,
        z: message.pose.pose.position.z 
      }
    })
    // Initialize camera stream after robot connects
    setCamera()
  })


  ros.value.on('error', (error) => 
  {
    console.log('not connected')
    connected.value = false
  })

  ros.value.on('close', () => 
  {
    console.log('closed')
    connected.value = false
    document.getElementById('divCamera').innerHTML = ''
  })
}

const send_step_command = (direction) => {
  // if(!ros.value || !connected.value)
  // {
  //   alert("Please connect to the robot first")
  // }

  let topic = new ROSLIB.Topic({
    ros: ros.value,
    name: '/cmd_vel',
    messageType: 'geometry_msgs/Twist'
  })
  // console.log(direction)
  const twist = {
          linear: {
            x: 0.0,
            y: 0.0,
            z: 0.0
          },
          angular: {
            x: 0.0,
            y: 0.0,
            z: 0.0
          }
        }
  switch(direction)
  {
    case 'forward':{
        twist.linear.x = 0.3
      }
      break
    case 'backward':{
        twist.linear.x = -0.3
      }
      break
    case 'left':{
        twist.angular.z = 0.8
      }
      break

    case 'right': {
        twist.angular.z =-0.8
      }

      break
    default:
      break
  }
  topic.publish(twist)
}

const send_velocity_command = (linear_x, angular_z) => {
  if(!ros.value || !connected.value)
  {
    alert("Please connect to the robot first")
  }

  let topic = new ROSLIB.Topic({
    ros: ros.value,
    name: '/cmd_vel',
    messageType: 'geometry_msgs/Twist'
  })
  const twist = {
          linear: {
            x: linear_x,
            y: 0.0,
            z: 0.0
          },
          angular: {
            x: 0.0,
            y: 0.0,
            z: -angular_z
          }
        }

  topic.publish(twist)
}

const setCamera = () => {
  if(!rosbridge_address.value || rosbridge_address.value==''){
    alert("Please enter the robot IP address first")
    return
  }
  
  // Extract IP from rosbridge address
  let robotIP = rosbridge_address.value
  try {
    if (robotIP.includes('//')) {
      robotIP = robotIP.split('//')[1].split(':')[0]
    }
  } catch (e) {
    alert("Invalid IP address format. Expected: ws://192.168.x.x:9090")
    return
  }
  console.log("robotIP", robotIP)
  let viewer = new MJPEGCANVAS.Viewer({
    divID: 'divCamera',
    host: robotIP,
    port: '8080',
    width: 640,
    height: 480,
    topic: '/image_raw'
  });
}

const call_service = () => {
  console.log("call service")
  let service_response = ''
  let service = new ROSLIB.Service({
    ros: ros.value,
    name: '/test_srv',
    serviceType: 'test_srv/srv/TestSrv',
  })

  let request = new ROSLIB.ServiceRequest({
    "message": {"data": "fuck you"}
  })

  service.callService(request, (result) => {
    service_response = JSON.stringify(result)
    console.log("service_response", service_response)
  }, (error) => {console.log("error", error)})
}

const startDrag = (e) => {
  isDragging.value = true
  dragZoneRect.value = document.getElementById('dragstartzone').getBoundingClientRect()
  doDrag(e)
}


const doDrag = (e) => {
  if (!isDragging.value) return
  
  const zone = document.getElementById('dragstartzone')
  if (!zone) return
  
  const rect = zone.getBoundingClientRect()
  const centerX = 100
  const centerY = 100
  const maxRadius = 75
  
  let x = e.clientX - rect.left - centerX
  let y = e.clientY - rect.top - centerY
  
  const distance = Math.sqrt(x * x + y * y)
  if (distance > maxRadius) {
    const angle = Math.atan2(y, x)
    x = Math.cos(angle) * maxRadius
    y = Math.sin(angle) * maxRadius
  }
  
  dragCircleStyle.value = {
    left: (50 + (x / 200) * 100) + '%',
    top: (50 + (y / 200) * 100) + '%'
  }
  
  // Send command based on joystick position
  // const angle = Math.atan2(y, x) * 180 / Math.PI
  // const magnitude = distance / maxRadius
  let max_velo = 0.3
  let max_angular = 0.3
  const x_mag = Math.sin(x * Math.PI/180) * max_velo
  const y_mag = -1 * Math.sin(y * Math.PI/180) * max_angular 

  
  console.log("angular", x_mag, "linear", y_mag)

  send_velocity_command(y_mag, x_mag)
}

const stopDrag = () => {
  isDragging.value = false
  dragCircleStyle.value = {
    left: '50%',
    top: '50%'
  }
  send_step_command('stop')
}

onMounted(() => {
  document.addEventListener('mouseup', stopDrag)
})

onUnmounted(() => {
  document.removeEventListener('mouseup', stopDrag)
  if(ros.value)
  {
    ros.value.close()
  }
})
</script>

<template>
  <header>
    <div>
      <h1>Hello from Robot Ignite Academy!</h1>
      <p>Let's connect our website to a ROS robot!</p>
    </div>
  </header>
  <main>
  <div>
    <p>IP Address:</p>
    <input
      v-model="rosbridge_address"
      placeholder="Enter the IP"
    />
    <button @click="connect_robot">Enter</button>
  </div>
  <div style="position: relative; width: 200px; height: 200px; margin: 0 auto;">
    <button style="position: absolute; top: 0; left: 50%; transform: translateX(-50%);" @click="() => send_step_command('forward')">forward</button>
    <button style="position: absolute; bottom: 0; left: 50%; transform: translateX(-50%);" @click="() => send_step_command('backward')">backward</button>
    <button style="position: absolute; top: 50%; left: 0; transform: translateY(-50%);" @click="() => send_step_command('left')">turn left</button>
    <button style="position: absolute; top: 50%; right: 0; transform: translateY(-50%);" @click="() => send_step_command('right')">turn right</button>
    <button style="position: absolute; top: 50%; right: 40%; transform: translateY(-50%);" @click="() => send_step_command('')">stop</button>
  </div>
  <div class="joystick-container">
    <div class="joystick-panel">
      <h2>Joystick Controller</h2>
      <hr>
      <p>Drag the joystick to control the robot</p>
      <div class="joystick-wrapper">
        <div id="dragstartzone" @mousedown="startDrag" @mousemove="doDrag" @mouseup="stopDrag">
          <div id="dragCircle" :style="dragCircleStyle"></div>
        </div>
      </div>
    </div>
  </div>
  <div class="odom-display">
    <h3>Robot Position:</h3>
    <p>X: {{ position.x.toFixed(2) }}</p>
    <p>Y: {{ position.y.toFixed(2) }}</p>
    <p>Z: {{ position.z.toFixed(2) }}</p>
  </div>
  <div>
    <div class="col-md-6 col-sm-6 text-center">
      <div id="divCamera"></div>
    </div>
  </div>
  <div style="position: relative; width: 100px; height: 100px; margin: 0 auto;">
    <button @click="call_service">call service</button>
  </div>
</main>
</template>

<style scoped>
.odom-display {
  margin-top: 20px;
  text-align: center;
  font-family: Arial, sans-serif;
}


.joystick-container {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 40px;
  margin-bottom: 40px;
}

.joystick-panel {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 10px;
  padding: 30px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  color: white;
  max-width: 400px;
}

.joystick-panel h2 {
  margin: 0 0 10px 0;
  font-size: 24px;
}

.joystick-panel hr {
  width: 100%;
  border: none;
  border-top: 2px solid rgba(255, 255, 255, 0.3);
  margin: 10px 0;
}

.joystick-panel p {
  margin: 10px 0 20px 0;
  font-size: 14px;
  opacity: 0.9;
}

.joystick-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
}

#dragstartzone {
  position: relative;
  width: 200px;
  height: 200px;
  background: radial-gradient(circle, rgba(255, 255, 255, 0.2) 0%, rgba(255, 255, 255, 0.05) 100%);
  border: 3px solid rgba(255, 255, 255, 0.4);
  border-radius: 50%;
  cursor: grab;
  user-select: none;
}

#dragstartzone:active {
  cursor: grabbing;
}

#dragCircle {
  position: absolute;
  width: 40px;
  height: 40px;
  background: white;
  border-radius: 50%;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.4);
  transition: none;
  transform: translate(-50%, -50%);
  pointer-events: none;
}

#divCamera {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 30px;
  margin-bottom: 30px;
  padding: 10px;
  background: #f0f0f0;
  border-radius: 10px;
  min-height: 480px;
}

#divCamera canvas {
  max-width: 100%;
  height: auto;
  border-radius: 5px;
}
</style>
