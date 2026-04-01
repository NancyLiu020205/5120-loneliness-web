<template>
  <div class="home-container">
    <div class="glass-panel">
      <h1 class="headline">
        <span class="text-dark">Embrace Community</span><br />
        <span class="text-orange">Right Here in Melbourne.</span>
      </h1>
      <p class="description">
        Growing older shouldn't mean feeling alone. We help Melbourne's seniors over 60 discover local activities, find companionship, and access mental support resources. Join a caring community and add more joy to your everyday life.
      </p>
      <div class="actions">
        <router-link to="/nearby-mental-support" class="btn btn-primary">Find Nearby Support</router-link>
        <router-link to="/my-routes" class="btn btn-secondary">Plan a Route</router-link>
      </div>
    </div>
  </div>
</template>

<style scoped>
.home-container {
  /* Ensure it fills the screen minus the navbar height */
  min-height: calc(100vh - 64px); 
  display: flex;
  justify-content: center;
  align-items: center;
  /* Sunset over water or relaxing city scenery */
  background-image: url('https://images.unsplash.com/photo-1545044846-e5b1cb6a6442?q=80&w=2000&auto=format&fit=crop');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  padding: 40px 20px;
}

.glass-panel {
  /* Glassmorphism effect */
  background: rgba(255, 255, 255, 0.85);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  border: 1px solid rgba(255, 255, 255, 0.6);
  border-radius: 20px;
  padding: 60px 50px;
  max-width: 850px;
  text-align: center;
  box-shadow: 0 10px 40px 0 rgba(0, 0, 0, 0.1);
}

.headline {
  /* Large, legible font for seniors */
  font-size: 3.25rem;
  font-weight: 800;
  line-height: 1.25;
  margin-bottom: 24px;
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
}

.text-dark {
  color: #2d3748;
}

.text-orange {
  /* Warm, welcoming orange */
  color: #ed8936;
}

.description {
  font-size: 1.35rem;
  color: #4a5568;
  line-height: 1.6;
  margin-bottom: 40px;
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
}

.actions {
  display: flex;
  gap: 24px;
  justify-content: center;
  flex-wrap: wrap;
}

.btn {
  padding: 16px 32px;
  font-size: 1.125rem;
  font-weight: 600;
  border-radius: 50px;
  text-decoration: none;
  transition: all 0.3s ease;
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
}

.btn-primary {
  background-color: #ed8936;
  color: #ffffff;
  border: 2px solid #ed8936;
}

.btn-primary:hover {
  background-color: #dd6b20;
  border-color: #dd6b20;
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(237, 137, 54, 0.2);
}

.btn-secondary {
  background-color: transparent;
  color: #dd6b20;
  border: 2px solid #ed8936;
}

.btn-secondary:hover {
  background-color: rgba(237, 137, 54, 0.05);
  transform: translateY(-2px);
}

/* Responsive adjustments */
@media (max-width: 768px) {
  .headline {
    font-size: 2.5rem;
  }
  .glass-panel {
    padding: 40px 24px;
    border-radius: 16px;
  }
  .description {
    font-size: 1.15rem;
  }
}
</style>
