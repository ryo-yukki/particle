<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Gather and Scatter Particles</title>
  <style>
    body { margin: 0; overflow: hidden; }
    canvas { display: block; }
  </style>
</head>
<body>
<canvas id="glcanvas"></canvas>
<script>
const canvas = document.getElementById("glcanvas");
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
const gl = canvas.getContext("webgl");

const NUM_PARTICLES = 300;
const positions = new Float32Array(NUM_PARTICLES * 2);
const velocities = new Float32Array(NUM_PARTICLES * 2);
const colors = new Float32Array(NUM_PARTICLES * 3);

// 初期化
for (let i = 0; i < NUM_PARTICLES; i++) {
  positions[i * 2] = (Math.random() * 2 - 1);
  positions[i * 2 + 1] = (Math.random() * 2 - 1);
  velocities[i * 2] = 0;
  velocities[i * 2 + 1] = 0;

  colors[i * 3] = Math.random();
  colors[i * 3 + 1] = Math.random();
  colors[i * 3 + 2] = Math.random();
}

let leader = {
  x: Math.random() * 2 - 1,
  y: Math.random() * 2 - 1,
  vx: 0,
  vy: 0
};

let mode = "gather";
let modeTimer = 0;
const MODE_DURATION = 80;

const vsSource = `
  attribute vec2 a_position;
  attribute vec3 a_color;
  varying vec3 v_color;
  void main() {
    gl_PointSize = 10.0;
    gl_Position = vec4(a_position, 0.0, 1.0);
    v_color = a_color;
  }
`;

const fsSource = `
  precision mediump float;
  varying vec3 v_color;
  void main() {
    gl_FragColor = vec4(v_color, 1.0);
  }
`;

function createShader(gl, type, source) {
  const shader = gl.createShader(type);
  gl.shaderSource(shader, source);
  gl.compileShader(shader);
  return shader;
}

function createProgram(gl, vs, fs) {
  const program = gl.createProgram();
  gl.attachShader(program, vs);
  gl.attachShader(program, fs);
  gl.linkProgram(program);
  return program;
}

const vs = createShader(gl, gl.VERTEX_SHADER, vsSource);
const fs = createShader(gl, gl.FRAGMENT_SHADER, fsSource);
const program = createProgram(gl, vs, fs);

const positionBuffer = gl.createBuffer();
const colorBuffer = gl.createBuffer();
const positionLoc = gl.getAttribLocation(program, "a_position");
const colorLoc = gl.getAttribLocation(program, "a_color");

function updateParticles() {
  // モード切り替え
  modeTimer++;
  if (modeTimer > MODE_DURATION) {
    modeTimer = 0;
    if (mode === "gather") {
      mode = "scatter";
      // ランダムに発散ベクトルを与える
      for (let i = 0; i < NUM_PARTICLES; i++) {
        velocities[i * 2] = (Math.random() - 0.5) * 0.10;
        velocities[i * 2 + 1] = (Math.random() - 0.5) * 0.05;
      }
    } else {
      mode = "gather";
    }
  }

  // リーダー：ランダムウォーク
  leader.vx += (Math.random() - 0.5) * 0.001;
  leader.vy += (Math.random() - 0.5) * 0.001;
  leader.vx *= 0.98;
  leader.vy *= 0.98;
  leader.x += leader.vx;
  leader.y += leader.vy;
  if (leader.x < -1 || leader.x > 1) leader.vx *= -1;
  if (leader.y < -1 || leader.y > 1) leader.vy *= -1;

  for (let i = 0; i < NUM_PARTICLES; i++) {
    const idx = i * 2;

    if (mode === "gather") {
      const dx = leader.x - positions[idx];
      const dy = leader.y - positions[idx + 1];
      const dist = Math.sqrt(dx * dx + dy * dy) + 0.001;
      const force = 0.005;
      velocities[idx] += (dx / dist) * force;
      velocities[idx + 1] += (dy / dist) * force;
    }

    velocities[idx] *= 0.95;
    velocities[idx + 1] *= 0.95;

    positions[idx] += velocities[idx];
    positions[idx + 1] += velocities[idx + 1];
  }

  positions[0] = leader.x;
  positions[1] = leader.y;
}

function render() {
  updateParticles();

  gl.viewport(0, 0, canvas.width, canvas.height);
  gl.clear(gl.COLOR_BUFFER_BIT);
  gl.useProgram(program);

  gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
  gl.bufferData(gl.ARRAY_BUFFER, positions, gl.DYNAMIC_DRAW);
  gl.enableVertexAttribArray(positionLoc);
  gl.vertexAttribPointer(positionLoc, 2, gl.FLOAT, false, 0, 0);

  gl.bindBuffer(gl.ARRAY_BUFFER, colorBuffer);
  gl.bufferData(gl.ARRAY_BUFFER, colors, gl.STATIC_DRAW);
  gl.enableVertexAttribArray(colorLoc);
  gl.vertexAttribPointer(colorLoc, 3, gl.FLOAT, false, 0, 0);

  gl.drawArrays(gl.POINTS, 0, NUM_PARTICLES);
  requestAnimationFrame(render);
}

gl.clearColor(0.02, 0.02, 0.05, 1.0);
render();
</script>
</body>
</html>
