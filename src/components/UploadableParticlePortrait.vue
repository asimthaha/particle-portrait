<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";

// --- Types ---
type ParticleAnimationState = "swirling" | number;
type PotentialParticle = { x: number; y: number; color: string };

//
// --- MODIFIED Particle Class ---
//
class Particle {
  // --- Configuration Variables ---
  static particleDensity: number = 3;
  static particleSize: number = 1.5;
  static easeFactor: number = 0.04;
  static maxCanvasWidth: number = 320; // This will be the max width
  static swirlRandomWalkSpeed: number = 0.8;

  // --- Instance Properties ---
  imageTargets: { x: number; y: number; color: string }[];
  targetX: number;
  targetY: number;
  x: number;
  y: number;
  color: string;
  speed: number;

  // Random walk properties
  vx: number;
  vy: number;

  constructor(
    imageTargets: { x: number; y: number; color: string }[],
    canvasWidth: number,
    canvasHeight: number
  ) {
    this.imageTargets = imageTargets;
    // FIX 1 (Line 39): Use optional chaining and a fallback color
    this.color = this.imageTargets[0]?.color || "white";
    this.targetX = 0;
    this.targetY = 0;
    this.x = Math.random() * canvasWidth;
    this.y = Math.random() * canvasHeight;
    this.speed = Math.random() * 0.05 + Particle.easeFactor;

    this.vx = (Math.random() - 0.5) * Particle.swirlRandomWalkSpeed;
    this.vy = (Math.random() - 0.5) * Particle.swirlRandomWalkSpeed;
  }

  // FIX 2 (Lines 54, 55, 56): Removed unused parameters
  update(
    canvasWidth: number,
    canvasHeight: number,
    animationState: ParticleAnimationState
  ): void {
    if (animationState === "swirling") {
      this.x += this.vx;
      this.y += this.vy;

      if (this.x > canvasWidth) this.x = 0;
      if (this.x < 0) this.x = canvasWidth;
      if (this.y > canvasHeight) this.y = 0;
      if (this.y < 0) this.y = canvasHeight;

      this.vx += (Math.random() - 0.5) * 0.1;
      this.vy += (Math.random() - 0.5) * 0.1;

      this.vx = Math.max(
        -Particle.swirlRandomWalkSpeed,
        Math.min(Particle.swirlRandomWalkSpeed, this.vx)
      );
      this.vy = Math.max(
        -Particle.swirlRandomWalkSpeed,
        Math.min(Particle.swirlRandomWalkSpeed, this.vy)
      );
    } else {
      const target = this.imageTargets[animationState];
      if (target) {
        this.targetX = target.x;
        this.targetY = target.y;
        this.color = target.color;
      }
      this.x += (this.targetX - this.x) * this.speed;
      this.y += (this.targetY - this.y) * this.speed;
    }
  }

  draw(ctx: CanvasRenderingContext2D): void {
    ctx.fillStyle = this.color;
    ctx.fillRect(this.x, this.y, Particle.particleSize, Particle.particleSize);
  }
}

// --- Vue Composition API ---

const canvasRef = ref<HTMLCanvasElement | null>(null);
const fileInputRef = ref<HTMLInputElement | null>(null);

let ctx: CanvasRenderingContext2D | null = null;
let particlesArray: Particle[] = [];
let animationFrameId: number | null = null;
let animationState: ParticleAnimationState = "swirling";

// --- FIX 3 (Lines 150-153): Removed the redundant loadImageData function ---

// --- Helper to extract particles from image data ---
const getPotentialParticles = (imageData: ImageData): PotentialParticle[] => {
  const particles: PotentialParticle[] = [];
  const data = imageData.data;
  const width = imageData.width;
  const height = imageData.height;

  for (let y = 0; y < height; y += Particle.particleDensity) {
    for (let x = 0; x < width; x += Particle.particleDensity) {
      const index = (y * width + x) * 4;
      const r = data[index];
      const g = data[index + 1];
      const b = data[index + 2];
      const a = data[index + 3];

      // FIX 4 (Lines 178, 180): Add guard for undefined pixel data
      if (
        r === undefined ||
        g === undefined ||
        b === undefined ||
        a === undefined
      ) {
        continue;
      }

      const color = `rgb(${r},${g},${b})`;
      const brightness = (r + g + b) / 3;

      if (a > 100 && brightness > 10) {
        particles.push({ x, y, color });
      }
    }
  }
  return particles;
};

// --- MODIFIED: Simplified initParticles ---
const initParticles = (potentialParticles: PotentialParticle[]) => {
  const canvas = canvasRef.value;
  if (!canvas) return;

  particlesArray = []; // Clear old particles
  const maxParticles = potentialParticles.length;
  console.log(`Creating ${maxParticles} total particles.`);

  potentialParticles.sort(() => 0.5 - Math.random());

  for (let i = 0; i < maxParticles; i++) {
    const pData = potentialParticles[i];

    // FIX 5 (Line 201): Add guard for pData
    if (!pData) continue;

    const imageTargets = [{ x: pData.x, y: pData.y, color: pData.color }];

    const particle = new Particle(imageTargets, canvas.width, canvas.height);

    particlesArray.push(particle);
  }
};

// --- MODIFIED: Animation Loop ---
const animate = (_time: number) => {
  const canvas = canvasRef.value;
  if (!ctx || !canvas) return;

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  for (let i = 0; i < particlesArray.length; i++) {
    const particle = particlesArray[i];

    // FIX 6 (Lines 230, 238): Add guard for particle
    if (!particle) continue;

    // FIX 7 (Line 230): Update call signature
    particle.update(canvas.width, canvas.height, animationState);
    particle.draw(ctx);
  }
  animationFrameId = requestAnimationFrame(animate);
};

// --- Main setup/resize function ---
const setupCanvas = () => {
  const canvas = canvasRef.value;
  if (!canvas) return;

  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
    animationFrameId = null;
  }

  const dpr = window.devicePixelRatio || 1;
  const newSize = canvas.parentElement
    ? canvas.parentElement.getBoundingClientRect().width
    : 0;
  const finalCssSize = Math.min(newSize, Particle.maxCanvasWidth);

  if (finalCssSize <= 0) {
    console.warn("Parent width is 0, waiting for resize.");
    return;
  }

  const finalBitmapSize = finalCssSize * dpr;
  canvas.width = finalBitmapSize;
  canvas.height = finalBitmapSize; // Start as a square
  canvas.style.width = `${finalCssSize}px`;
  canvas.style.height = `${finalCssSize}px`;
  canvas.style.aspectRatio = `1 / 1`;

  ctx = canvas.getContext("2d", { willReadFrequently: true });

  particlesArray = [];
  animationState = "swirling";
};

// --- CRITICAL MODIFICATION: Handler for file input change ---
const handleFileChange = async (event: Event) => {
  const target = event.target as HTMLInputElement;
  const file = target.files?.[0];
  const canvas = canvasRef.value;
  if (!file || !canvas) return;

  animationState = "swirling";

  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
    animationFrameId = null;
  }

  const imageUrl = URL.createObjectURL(file);
  const image = new Image();
  image.src = imageUrl;

  image.onload = async () => {
    try {
      // --- Step 2: Calculate new canvas size ---
      const dpr = window.devicePixelRatio || 1;
      const imgAspectRatio = image.width / image.height;
      const maxWidth = Particle.maxCanvasWidth;
      const finalCssWidth = maxWidth;
      const finalCssHeight = finalCssWidth / imgAspectRatio;
      const finalBitmapWidth = finalCssWidth * dpr;
      const finalBitmapHeight = finalCssHeight * dpr;

      // --- Step 3: Resize the canvas ---
      canvas.width = finalBitmapWidth;
      canvas.height = finalBitmapHeight;
      canvas.style.width = `${finalCssWidth}px`;
      canvas.style.height = `${finalCssHeight}px`;
      canvas.style.aspectRatio = `${finalCssWidth} / ${finalCssHeight}`;

      // --- Step 4: Get context (and check it) ---
      ctx = canvas.getContext("2d", { willReadFrequently: true });
      // This check ensures 'ctx' is not null for the lines below
      if (!ctx) throw new Error("Could not get 2D context after resize");

      // --- Step 5: Get Image Data (Logic moved from loadImageData) ---
      // FIX 8 (Lines 150-153): 'ctx' is now guaranteed to be non-null
      const canvasAspectRatio = canvas.width / canvas.height;
      let dWidth, dHeight, dx, dy;

      if (imgAspectRatio > canvasAspectRatio) {
        dWidth = canvas.width;
        dHeight = dWidth / imgAspectRatio;
        dx = 0;
        dy = (canvas.height - dHeight) / 2;
      } else {
        dHeight = canvas.height;
        dWidth = dHeight * imgAspectRatio;
        dy = 0;
        dx = (canvas.width - dWidth) / 2;
      }

      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.drawImage(image, dx, dy, dWidth, dHeight);
      const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      URL.revokeObjectURL(imageUrl); // Clean up memory

      // --- Step 6: Initialize particles ---
      console.log("Initializing particles...");
      initParticles(getPotentialParticles(imageData));

      // --- Step 7: Start animation ---
      console.log("Starting animation...");
      animate(0);

      setTimeout(() => {
        console.log("Assembling portrait.");
        animationState = 0;
      }, 2000);
    } catch (error) {
      console.error("Failed to load and process image:", error);
      if (ctx && canvas) {
        ctx.fillStyle = "red";
        ctx.font = "16px Arial";
        ctx.textAlign = "center";
        ctx.fillText(
          "Error loading image.",
          canvas.width / 2,
          canvas.height / 2
        );
      }
      URL.revokeObjectURL(imageUrl); // Clean up on error
    }
  };

  image.onerror = () => {
    console.error("Image failed to load");
    URL.revokeObjectURL(imageUrl);
  };
};

// --- Handler for upload button click ---
const handleUploadClick = () => {
  fileInputRef.value?.click();
};

// --- Lifecycle Hooks ---
onMounted(() => {
  setupCanvas();
  window.addEventListener("resize", setupCanvas);
});

onUnmounted(() => {
  window.removeEventListener("resize", setupCanvas);
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
});
</script>

<template>
  <div class="container">
    <div class="button-container">
      <button @click="handleUploadClick" class="upload-button">
        Upload Photo
      </button>
      <input
        type="file"
        ref="fileInputRef"
        @change="handleFileChange"
        accept="image/*"
        style="display: none"
      />
    </div>
    <div>
      <canvas ref="canvasRef" />
    </div>
  </div>
</template>

<style>
/* Global styles */
body {
  margin: 0;
  padding: 0;
  overflow: hidden;
  background-color: #111;
}
</style>

<style scoped>
/* Component-specific styles */
.container {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  position: relative;
}

/* Add a transition to the canvas container for smoother resizing */
.container > div {
  transition: all 0.3s ease-in-out;
}

.button-container {
  position: absolute;
  top: 20px;
  right: 20px;
  z-index: 10;
}

.upload-button {
  padding: 10px 20px;
  font-size: 16px;
  color: white;
  background: #007bff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.upload-button:hover {
  background: #0056b3;
}
</style>
