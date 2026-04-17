<template>
  <div class="home-wrapper">
    <!-- Hero Section -->
    <section class="hero-section">
      <div class="glass-panel main-intro">
        <h1 class="headline">
          <span class="text-dark">Embrace Community</span><br />
          <span class="text-green">Right Here in Melbourne.</span>
        </h1>
        <p class="description">
          Growing older shouldn't mean feeling alone. We help Melbourne seniors over 60 discover
          local activities and access mental support resources.
        </p>
        <router-link to="/my-routes" class="btn-solid btn-hero-route btn-large">
          Start Your Customized Route
        </router-link>
      </div>
    </section>

    <!-- Feature Sections -->
    <div class="features-container">
      <!-- Section 1: Customized Routes -->
      <section id="custom-routes" class="feature-section routes-bg">
        <div class="feature-card">
          <h2 class="feature-title">Customized Routes</h2>
          <p class="feature-description">
            Plan a journey that fits your needs perfectly. Whether you prefer a quiet walk with more
            shade, or need to find public toilets and rest benches along the way, our smart routing
            tool helps you navigate Melbourne with comfort and confidence.
          </p>
          <ul class="feature-list">
            <li>
              🌲 <strong>Shade Optimization:</strong> Choose routes with more tree canopy for a
              cooler walk.
            </li>
            <li>
              🚻 <strong>Accessibility First:</strong> Locate public toilets and rest benches on
              your path.
            </li>
            <li>🚶 <strong>Your Pace:</strong> Select from Walking, Cycling, or Transit modes.</li>
          </ul>
          <router-link to="/my-routes" class="btn-solid btn-primary btn-large">
            Plan Your Route Now
          </router-link>
        </div>
      </section>

      <!-- Section 2: Mental Support -->
      <section id="mental-support" class="feature-section mental-bg">
        <div class="feature-card">
          <h2 class="feature-title">Mental Health Support</h2>
          <p class="feature-description">
            You're never alone in Melbourne. Easily locate nearby mental health counseling centers
            and community support resources tailored for seniors. Our platform provides real-time
            access to professional help and caring communities, ensuring you have the support you
            need, whenever you need it.
          </p>
          <ul class="feature-list">
            <li>
              📍 <strong>Nearby Resources:</strong> Find counseling centers closest to your current
              location.
            </li>
            <li>
              🗺️ <strong>Easy Directions:</strong> Open map guidance quickly and travel with fewer
              detours.
            </li>
          </ul>
          <router-link to="/nearby-mental-support" class="btn-solid btn-secondary btn-large">
            Find Support Nearby
          </router-link>
        </div>
      </section>
    </div>

    <!-- Footer Credit -->
    <footer class="home-footer">
      <p>&copy; 2026 Melbourne Senior Community Support. All rights reserved.</p>
    </footer>
  </div>
</template>

<script setup>
// No script logic needed for now, but keeping setup for future use
</script>

<style scoped>
.home-wrapper {
  position: relative;
  overflow-x: hidden;
  scroll-behavior: smooth;
  background-color: #ffffff;
  background-image:
    radial-gradient(at 0% 0%, rgba(34, 197, 94, 0.05) 0, transparent 50%),
    radial-gradient(at 50% 0%, rgba(59, 130, 246, 0.05) 0, transparent 50%);
  background-size: cover;
  background-position: center;
  background-attachment: fixed;
  font-family:
    'Inter',
    system-ui,
    -apple-system,
    sans-serif;
}

.hero-section {
  min-height: calc(100vh - 80px);
  display: flex;
  justify-content: center;
  align-items: center;
  background-image:
    linear-gradient(135deg, rgba(255, 255, 255, 0.4) 0%, rgba(255, 255, 255, 0.1) 100%),
    url('@/assets/hero-bg.png');
  background-size: cover;
  background-position: center;
  position: relative;
  border-bottom: 1px solid rgba(0, 0, 0, 0.05);
}

.main-intro {
  max-width: 850px;
  margin: 0 20px;
  animation: fadeIn 1s ease-out;
}

.glass-panel {
  background: rgba(255, 255, 255, 0.75);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.5);
  border-radius: 40px;
  padding: 60px 50px;
  text-align: center;
  box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.1);
}

.headline {
  font-size: 3.75rem;
  font-weight: 800;
  line-height: 1.2;
  margin-bottom: 24px;
  letter-spacing: -0.02em;
}

.text-dark {
  color: #1e293b;
}
.text-green {
  color: #166534;
}

.description {
  font-size: 1.5rem;
  color: #475569;
  line-height: 1.6;
  margin-bottom: 36px;
  max-width: 700px;
  margin-left: auto;
  margin-right: auto;
}

/* --- Features Sections --- */
.features-container {
  display: flex;
  flex-direction: column;
}

.feature-section {
  padding: 84px 20px;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 68vh;
}

.routes-bg {
  background: linear-gradient(135deg, #f0fdf4 0%, #dcfce7 100%);
  position: relative;
  overflow: hidden;
}

.routes-bg::after {
  content: '';
  position: absolute;
  right: -5%;
  top: 10%;
  width: 40%;
  height: 80%;
  background-image: url('https://images.unsplash.com/photo-1519331379826-f10be5486c6f?q=80&w=1000&auto=format&fit=crop');
  background-size: cover;
  border-radius: 40px;
  opacity: 0.1;
  transform: rotate(5deg);
  pointer-events: none;
}

.mental-bg {
  background: linear-gradient(135deg, #eff6ff 0%, #dbeafe 100%);
  position: relative;
  overflow: hidden;
}

.mental-bg::after {
  content: '';
  position: absolute;
  left: -5%;
  bottom: 5%;
  width: 35%;
  height: 70%;
  background-image: url('https://images.unsplash.com/photo-1573497019940-1c28c88b4f3e?q=80&w=1000&auto=format&fit=crop');
  background-size: cover;
  border-radius: 40px;
  opacity: 0.1;
  transform: rotate(-3deg);
  pointer-events: none;
}

.feature-card {
  max-width: 900px;
  background: white;
  padding: 34px 46px 40px;
  border-radius: 40px;
  box-shadow: 0 20px 40px -10px rgba(0, 0, 0, 0.05);
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  text-align: left;
  z-index: 1;
  transition: transform 0.3s ease;
}

.feature-card:hover {
  transform: translateY(-5px);
}

.feature-title {
  font-size: 3rem;
  font-weight: 800;
  color: #1e293b;
  margin-bottom: 20px;
}

.feature-description {
  font-size: 1.25rem;
  color: #475569;
  line-height: 1.7;
  margin-bottom: 32px;
}

.feature-list {
  list-style: none;
  padding: 0;
  margin-bottom: 40px;
}

.feature-list li {
  font-size: 1.125rem;
  color: #334155;
  margin-bottom: 16px;
  display: flex;
  align-items: center;
  gap: 12px;
}

.btn-large {
  padding: 18px 40px;
  font-size: 1.125rem;
  letter-spacing: 0.01em;
}

.btn-solid {
  display: inline-block;
  color: white;
  border: none;
  border-radius: 50px;
  font-weight: 700;
  text-decoration: none;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  text-align: center;
}

.btn-primary {
  background-color: #22c55e;
  box-shadow: 0 10px 20px -5px rgba(34, 197, 94, 0.45);
}

.btn-primary:hover {
  background-color: #16a34a;
  transform: translateY(-4px);
  box-shadow: 0 15px 25px -5px rgba(22, 163, 74, 0.5);
}

.btn-hero-route {
  margin-top: 24px;
  background-color: #22c55e;
  box-shadow: 0 10px 20px -5px rgba(34, 197, 94, 0.45);
}

.btn-hero-route:hover {
  background-color: #16a34a;
  transform: translateY(-4px);
  box-shadow: 0 15px 25px -5px rgba(22, 163, 74, 0.5);
}

.btn-secondary {
  background-color: #3b82f6;
  box-shadow: 0 10px 20px -5px rgba(59, 130, 246, 0.4);
}

.btn-secondary:hover {
  background-color: #2563eb;
  transform: translateY(-4px);
  box-shadow: 0 15px 25px -5px rgba(59, 130, 246, 0.5);
}

.home-footer {
  padding: 60px 40px;
  text-align: center;
  background-color: #0f172a;
  color: #64748b;
  font-size: 1rem;
}

/* --- Animations --- */
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* --- Responsive adjustments --- */
@media (max-width: 1024px) {
  .headline {
    font-size: 3rem;
  }
  .feature-card {
    padding: 40px;
  }
  .routes-bg::after,
  .mental-bg::after {
    display: none;
  }
}

@media (max-width: 768px) {
  .home-wrapper {
    background-attachment: scroll;
  }
  .headline {
    font-size: 2.25rem;
  }
  .glass-panel {
    padding: 24px 12px;
    border-radius: 0;
  }
  .description {
    font-size: 1.125rem;
    margin-bottom: 30px;
  }
  .feature-title {
    font-size: 2rem;
  }
  .feature-card {
    padding: 24px 20px 28px;
    border-radius: 24px;
  }
  .feature-description {
    font-size: 1.05rem;
  }
  .btn-large {
    width: 100%;
    box-sizing: border-box;
  }
  .feature-list li {
    font-size: 1rem;
  }
}
</style>
