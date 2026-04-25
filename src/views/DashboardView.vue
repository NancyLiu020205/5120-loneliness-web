<template>
  <div class="dashboard">
    <div class="dashboard-inner">
      <header class="hero">
        <h1 class="hero-title">Community Health Insights</h1>
        <p class="hero-subtitle">
          Research-based information about social connection, mental health, and wellbeing among
          older Australians.
        </p>
      </header>

      <section class="section section--mental">
        <h2 class="section-title">This is a shared social issue</h2>
        <article class="card issue-card">
          <div class="issue-left">
            <div class="stat-box">
              <div class="stat-big">Almost 1 in 3</div>
              <div class="stat-small">Australians report moderate loneliness</div>
            </div>
            <div class="stat-box">
              <div class="stat-big">More than 1 in 6</div>
              <div class="stat-small">report severe loneliness</div>
            </div>
            <p class="source-text">Source: ELT 2023</p>
          </div>
          <div class="issue-right">
            <div class="donut-chart">
              <div v-for="item in sharedIssueBars" :key="item.label" class="donut-item">
                <div class="donut-svg-wrap">
                  <svg class="donut-svg" viewBox="0 0 140 140" aria-hidden="true">
                    <circle class="donut-track" cx="70" cy="70" r="50" />
                    <circle
                      class="donut-value"
                      cx="70"
                      cy="70"
                      r="50"
                      :style="getDonutStyle(item.value)"
                      transform="rotate(-90 70 70)"
                    />
                  </svg>
                  <button
                    class="donut-center"
                    type="button"
                    :title="`${item.label}: ${item.value}%`"
                    @mouseenter="showTooltip($event, `${item.label}: ${item.value}%`)"
                    @mousemove="showTooltip($event, `${item.label}: ${item.value}%`)"
                    @mouseleave="hideTooltip"
                  >
                    <span class="donut-percent">{{ item.value }}%</span>
                    <span class="donut-label">{{ item.label }}</span>
                  </button>
                </div>
              </div>
            </div>
            <p class="chart-note">
              Loneliness is commonly reported across the population and is not limited to a small
              group of individuals.
            </p>
          </div>
        </article>
      </section>

      <section class="section section--impact">
        <h2 class="section-title">Loneliness across different age groups</h2>
        <p class="section-subtitle">
          While loneliness is widely experienced, research shows that its prevalence varies across
          different age groups.
        </p>
        <article class="card age-card">
          <p class="source-text">Source: Ending Loneliness Together (2023)</p>
          <div class="v-chart-wrap">
            <div class="v-y-axis">
              <span>40%</span>
              <span>30%</span>
              <span>20%</span>
              <span>10%</span>
              <span>0%</span>
            </div>

            <div class="v-chart-panel">
              <div class="v-chart">
                <div class="v-grid-line" />
                <div class="v-grid-line" />
                <div class="v-grid-line" />
                <div class="v-grid-line" />
                <div class="v-grid-line" />

                <div
                  v-for="(item, index) in ageGroupBars"
                  :key="item.age"
                  class="v-col"
                  :class="{ 'is-active': hoveredAgeIndex === index }"
                  @mouseenter="onAgeBarEnter(index)"
                  @mouseleave="onAgeBarLeave"
                >
                  <div class="v-track">
                    <div class="v-bar" :style="{ height: `${(item.value / 40) * 100}%` }">
                      <span class="v-bar-value">{{ item.value }}%</span>
                    </div>
                  </div>

                  <div v-if="hoveredAgeIndex === index" class="v-inline-tooltip">
                    <p class="v-tooltip-title">{{ item.age }}</p>
                    <p class="v-tooltip-line">
                      <span class="v-tooltip-dot" />
                      <span>Loneliness</span>
                      <strong>{{ item.value }}</strong>
                    </p>
                  </div>
                </div>
              </div>

              <div class="v-x-axis">
                <span v-for="item in ageGroupBars" :key="`age-${item.age}`" class="v-age">
                  {{ item.age }}
                </span>
              </div>
            </div>
          </div>
          <p class="chart-note">
            Higher levels are reported among younger adults, but loneliness is present across all
            stages of life.
          </p>
          <p class="chart-note">
            Although reported levels are lower in older age groups, loneliness remains a relevant
            issue due to its impact on health and wellbeing.
          </p>
        </article>
      </section>

      <section class="section">
        <h2 class="section-title">Mental health and wellbeing context</h2>
        <p class="section-subtitle">
          This section looks beyond loneliness rates to show how mental health and local wellbeing
          relate to social connection in later life.
        </p>
        <div class="two-col">
          <article class="card mental-chart-card">
            <h3 class="card-title">National mental health conditions among older age groups</h3>
            <h4 class="mini-title">12-month anxiety disorder by age and sex</h4>
            <div class="group-chart-wrap">
              <div class="group-y-axis group-y-axis--20">
                <span>20%</span>
                <span>15%</span>
                <span>10%</span>
                <span>5%</span>
                <span>0%</span>
              </div>
              <div class="group-chart-panel">
                <div class="group-chart group-chart--20">
                  <div class="group-grid-line" />
                  <div class="group-grid-line" />
                  <div class="group-grid-line" />
                  <div class="group-grid-line" />
                  <div class="group-grid-line" />

                  <div
                    v-for="(item, index) in anxietyBars"
                    :key="`anxiety-${item.age}`"
                    class="group-col"
                    :class="{ 'is-active': hoveredAnxietyIndex === index }"
                    @mouseenter="onAnxietyBarEnter(index)"
                    @mouseleave="onAnxietyBarLeave"
                  >
                    <div class="pair-track">
                      <div
                        class="pair-bar pair-bar--male"
                        :style="{ height: `${(item.male / 20) * 100}%` }"
                      >
                        <span class="pair-value">{{ item.male }}%</span>
                      </div>
                      <div
                        class="pair-bar pair-bar--female"
                        :style="{ height: `${(item.female / 20) * 100}%` }"
                      >
                        <span class="pair-value">{{ item.female }}%</span>
                      </div>
                    </div>
                    <div v-if="hoveredAnxietyIndex === index" class="group-inline-tooltip">
                      <p class="group-tooltip-title">{{ item.age }}</p>
                      <p class="group-tooltip-line">
                        <span class="group-tooltip-dot group-tooltip-dot--male" />
                        <span>Male</span>
                        <strong>{{ formatTooltipValue(item.male) }}</strong>
                      </p>
                      <p class="group-tooltip-line">
                        <span class="group-tooltip-dot group-tooltip-dot--female" />
                        <span>Female</span>
                        <strong>{{ formatTooltipValue(item.female) }}</strong>
                      </p>
                    </div>
                  </div>
                </div>
                <div class="group-x-axis">
                  <span
                    v-for="item in anxietyBars"
                    :key="`anxiety-age-${item.age}`"
                    class="group-label"
                  >
                    {{ item.age }}
                  </span>
                </div>
              </div>
            </div>
            <div class="legend legend--chart">
              <span><i class="legend-dot male" /> Male</span>
              <span><i class="legend-dot female" /> Female</span>
            </div>

            <h4 class="mini-title">12-month mental disorders by age and sex</h4>
            <div class="group-chart-wrap">
              <div class="group-y-axis group-y-axis--25">
                <span>25%</span>
                <span>20%</span>
                <span>15%</span>
                <span>10%</span>
                <span>5%</span>
                <span>0%</span>
              </div>
              <div class="group-chart-panel">
                <div class="group-chart group-chart--25">
                  <div class="group-grid-line" />
                  <div class="group-grid-line" />
                  <div class="group-grid-line" />
                  <div class="group-grid-line" />
                  <div class="group-grid-line" />
                  <div class="group-grid-line" />

                  <div
                    v-for="(item, index) in disorderBars"
                    :key="`disorder-${item.age}`"
                    class="group-col"
                    :class="{ 'is-active': hoveredDisorderIndex === index }"
                    @mouseenter="onDisorderBarEnter(index)"
                    @mouseleave="onDisorderBarLeave"
                  >
                    <div class="pair-track">
                      <div
                        class="pair-bar pair-bar--male"
                        :style="{ height: `${(item.male / 25) * 100}%` }"
                      >
                        <span class="pair-value">{{ item.male }}%</span>
                      </div>
                      <div
                        class="pair-bar pair-bar--female"
                        :style="{ height: `${(item.female / 25) * 100}%` }"
                      >
                        <span class="pair-value">{{ item.female }}%</span>
                      </div>
                    </div>
                    <div v-if="hoveredDisorderIndex === index" class="group-inline-tooltip">
                      <p class="group-tooltip-title">{{ item.age }}</p>
                      <p class="group-tooltip-line">
                        <span class="group-tooltip-dot group-tooltip-dot--male" />
                        <span>Male</span>
                        <strong>{{ formatTooltipValue(item.male) }}</strong>
                      </p>
                      <p class="group-tooltip-line">
                        <span class="group-tooltip-dot group-tooltip-dot--female" />
                        <span>Female</span>
                        <strong>{{ formatTooltipValue(item.female) }}</strong>
                      </p>
                    </div>
                  </div>
                </div>
                <div class="group-x-axis">
                  <span
                    v-for="item in disorderBars"
                    :key="`disorder-age-${item.age}`"
                    class="group-label"
                  >
                    {{ item.age }}
                  </span>
                </div>
              </div>
            </div>

            <div class="legend">
              <span><i class="legend-dot male" /> Male</span>
              <span><i class="legend-dot female" /> Female</span>
            </div>
            <p class="chart-note">
              These figures show that mental health remains part of the later-life wellbeing
              picture, alongside social connection and community belonging.
            </p>
            <p class="source-text">
              Source: Australian Bureau of Statistics, National Study of Mental Health and Wellbeing
              2020-2022
            </p>
          </article>

          <article class="card local-wellbeing-card">
            <h3 class="card-title">Local wellbeing indicators for Melbourne residents aged 65+</h3>
            <div class="indicator-box">
              <p class="indicator-value">72.4%</p>
              <p class="indicator-label">Self-reported mental health rated excellent/very good</p>
            </div>
            <div class="indicator-box">
              <p class="indicator-value">73.0 / 100</p>
              <p class="indicator-label">Satisfaction with feeling part of the community</p>
            </div>
            <p class="chart-note">
              These local indicators describe self-reported wellbeing and community belonging,
              providing helpful context for understanding mental health and social connection among
              Melbourne residents aged 65+.
            </p>
            <p class="source-text">
              Source: City of Melbourne, Social Indicators for City of Melbourne Residents 2023
            </p>
          </article>
        </div>
      </section>

      <section class="section">
        <h2 class="section-title">Long-term loneliness affects more than mood</h2>
        <div class="impact-grid">
          <article v-for="item in impactCards" :key="item.title" class="card impact-card">
            <div class="impact-icon" aria-hidden="true">{{ item.icon }}</div>
            <p class="impact-value">{{ item.value }}</p>
            <h3 class="impact-title">{{ item.title }}</h3>
            <p class="impact-desc">{{ item.desc }}</p>
            <p class="source-text">Source: ELT Report</p>
          </article>
        </div>

        <article class="card isolation-layout">
          <section class="isolation-left">
            <h3 class="isolation-title">The Weight of Isolation</h3>
            <p class="isolation-intro">
              According to extensive meta-analyses led by Dr. Julianne Holt-Lunstad, chronic
              loneliness increases the risk of premature death significantly. The biological toll on
              the human body is staggering, equivalent to well-known health hazards.
            </p>

            <div class="isolation-fact-list">
              <div class="isolation-fact-item">
                <span class="fact-icon fact-icon--warning" aria-hidden="true">
                  <svg viewBox="0 0 24 24" class="fact-icon-svg">
                    <path d="M12 2.5L22 20.5H2L12 2.5Z" fill="currentColor" />
                    <rect x="11" y="8" width="2" height="7" fill="#ffffff" />
                    <rect x="11" y="16.5" width="2" height="2" fill="#ffffff" />
                  </svg>
                </span>
                <div>
                  <h4>Equivalent to 15 Cigarettes</h4>
                  <p>The mortality impact mirrors smoking 15 cigarettes daily.</p>
                </div>
              </div>

              <div class="isolation-fact-item">
                <span class="fact-icon fact-icon--health" aria-hidden="true">
                  <svg viewBox="0 0 24 24" class="fact-icon-svg">
                    <rect x="1.5" y="2.5" width="21" height="19" rx="3.4" fill="currentColor" />
                    <path
                      d="M4.3 12.2H8.1L10.1 8L12.8 15.3L14.9 11.2H19.8"
                      stroke="#ffffff"
                      stroke-width="1.8"
                      stroke-linecap="round"
                      stroke-linejoin="round"
                      fill="none"
                    />
                  </svg>
                </span>
                <div>
                  <h4>Worse than Obesity</h4>
                  <p>Isolation poses a higher risk than physical inactivity and obesity.</p>
                </div>
              </div>
            </div>
          </section>

          <section class="isolation-right" aria-label="Impact orbit">
            <div class="orbit-visual">
              <span class="orbit-ring orbit-ring--outer" aria-hidden="true" />
              <span class="orbit-ring orbit-ring--middle" aria-hidden="true" />
              <span class="orbit-ring orbit-ring--inner" aria-hidden="true" />
              <div class="orbit-core" aria-hidden="true">
                <span class="home-icon">⌂</span>
              </div>

              <span class="orbit-label orbit-label--top">Social support</span>
              <span class="orbit-label orbit-label--top-right">Belonging</span>
              <span class="orbit-label orbit-label--bottom-right">Physical health</span>
              <span class="orbit-label orbit-label--bottom">Mental wellbeing</span>
              <span class="orbit-label orbit-label--bottom-left">Cognitive resilience</span>
              <span class="orbit-label orbit-label--top-left">Quality of life</span>
            </div>
          </section>
        </article>

        <article class="card community-cta-card">
          <p class="community-cta-highlight">
            Small community interactions can help rebuild resilience.
          </p>
          <button class="community-cta-button" type="button" @click="goToEventPage">
            Explore nearby low-pressure activities
          </button>
          <p class="community-cta-desc">
            Gentle community contact can strengthen connection, confidence, and belonging.
          </p>
        </article>
      </section>
    </div>

    <div
      v-if="tooltip.visible"
      class="chart-tooltip"
      :style="{ left: `${tooltip.x}px`, top: `${tooltip.y}px` }"
    >
      {{ tooltip.text }}
    </div>
  </div>
</template>

<script setup>
import { reactive, ref } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()

const sharedIssueBars = [
  { label: 'Moderate loneliness', value: 32 },
  { label: 'Severe loneliness', value: 17.5 },
]

const ageGroupBars = [
  { age: '18-24', value: 38 },
  { age: '25-34', value: 32 },
  { age: '35-44', value: 35 },
  { age: '45-54', value: 36 },
  { age: '55-64', value: 34 },
  { age: '65-74', value: 26 },
  { age: '75+', value: 13 },
]

const anxietyBars = [
  { age: '55-64', male: 11.3, female: 16.8 },
  { age: '65-74', male: 6.3, female: 10.9 },
  { age: '75-85', male: 4.0, female: 5.5 },
]

const disorderBars = [
  { age: '55-64', male: 16.0, female: 19.6 },
  { age: '65-74', male: 9.6, female: 13.8 },
  { age: '75-85', male: 4.9, female: 6.1 },
]

const impactCards = [
  {
    icon: '❤',
    value: '4.6x',
    title: 'Depression',
    desc: 'Moderate loneliness was associated with substantially higher odds of depression in national survey data.',
  },
  {
    icon: '👥',
    value: '4.1x',
    title: 'Social anxiety',
    desc: 'Higher loneliness levels were associated with greater likelihood of social anxiety.',
  },
  {
    icon: '☹',
    value: '5.2x',
    title: 'Poor Wellbeing',
    desc: 'People reporting loneliness were more likely to report poorer wellbeing and lower quality of life.',
  },
]

const tooltip = reactive({
  visible: false,
  text: '',
  x: 0,
  y: 0,
})

const showTooltip = (event, text) => {
  tooltip.visible = true
  tooltip.text = text
  tooltip.x = event.clientX + 14
  tooltip.y = event.clientY + 14
}

const hideTooltip = () => {
  tooltip.visible = false
}

const hoveredAgeIndex = ref(null)
const onAgeBarEnter = (index) => {
  hoveredAgeIndex.value = index
}
const onAgeBarLeave = () => {
  hoveredAgeIndex.value = null
}

const goToEventPage = () => {
  router.push({ name: 'discover-nearby-places' })
}

const hoveredAnxietyIndex = ref(null)
const onAnxietyBarEnter = (index) => {
  hoveredAnxietyIndex.value = index
}
const onAnxietyBarLeave = () => {
  hoveredAnxietyIndex.value = null
}

const hoveredDisorderIndex = ref(null)
const onDisorderBarEnter = (index) => {
  hoveredDisorderIndex.value = index
}
const onDisorderBarLeave = () => {
  hoveredDisorderIndex.value = null
}

const formatTooltipValue = (value) => value.toFixed(1).replace(/\.0$/, '')

const donutRadius = 50
const donutCircumference = 2 * Math.PI * donutRadius
const getDonutStyle = (value) => ({
  strokeDasharray: `${(value / 100) * donutCircumference} ${donutCircumference}`,
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap');

.dashboard {
  --bg: #ffffff;
  --surface: #ffffff;
  --surface-soft: #eef6f0;
  --line: #d8e8dd;
  --text-main: #24363a;
  --text-muted: #5e7276;
  --green-strong: #2e7a58;
  --green-mid: #4f9c77;
  --green-light: #cfe9d7;
  --blue-bar: #3f78ab;
  --blue-bar-dark: #2f6493;
  --male: #2f7f68;
  --female: #6a8f4f;
  --kpi-bg: #e4f3e8;
  --warning-bg: #fff4dc;

  background: var(--bg);
  min-height: calc(100vh - 60px);
  padding: 34px 20px 56px;
  font-family: 'Inter', system-ui, sans-serif;
  color: var(--text-main);
}

.dashboard-inner {
  max-width: 1200px;
  margin: 0 auto;
}

.hero {
  text-align: center;
  margin-bottom: 30px;
}

.hero-title {
  margin: 0 0 10px;
  font-size: clamp(2rem, 4vw, 2.8rem);
  font-weight: 800;
  color: var(--green-strong);
}

.hero-subtitle {
  margin: 0 auto;
  max-width: 700px;
  color: var(--text-muted);
  line-height: 1.6;
}

.section {
  margin-top: 34px;
}

.section-title {
  margin: 0;
  font-size: clamp(1.55rem, 2.2vw, 2rem);
  font-weight: 700;
}

.section-subtitle {
  margin: 10px 0 16px;
  color: var(--text-muted);
  line-height: 1.65;
}

.card {
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 14px;
  box-shadow: 0 1px 3px rgba(32, 57, 43, 0.08);
}

.issue-card {
  margin-top: 14px;
  padding: 24px;
  display: grid;
  gap: 20px;
  grid-template-columns: 1fr;
}

.issue-left {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.stat-box {
  background: #e8f0ec;
  border-radius: 10px;
  padding: 20px 18px;
  text-align: center;
}

.stat-big {
  font-size: clamp(1.7rem, 2vw, 2.2rem);
  font-weight: 800;
  color: #2f6f7f;
}

.stat-small {
  margin-top: 6px;
  color: #516972;
}

.donut-chart {
  display: flex;
  flex-wrap: wrap;
  gap: 18px;
  margin-top: 8px;
}

.donut-item {
  flex: 1;
  min-width: 180px;
  display: flex;
  justify-content: center;
}

.donut-svg-wrap {
  position: relative;
  width: 160px;
  height: 160px;
}

.donut-svg {
  width: 100%;
  height: 100%;
}

.donut-track,
.donut-value {
  fill: none;
  stroke-width: 16;
}

.donut-track {
  stroke: #dde9e1;
}

.donut-value {
  stroke: #3f78ab;
  stroke-linecap: round;
}

.donut-center {
  position: absolute;
  inset: 24px;
  border: 0;
  border-radius: 999px;
  background: #ffffff;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  cursor: pointer;
  padding: 0 8px;
}

.donut-percent {
  font-size: 1.4rem;
  font-weight: 800;
  color: #2f6f7f;
}

.donut-label {
  margin-top: 4px;
  font-size: 0.78rem;
  color: #5d7174;
  line-height: 1.25;
}

.age-card {
  margin-top: 16px;
  padding: 18px 18px 14px;
}

.v-chart-wrap {
  margin-top: 8px;
  display: grid;
  grid-template-columns: 58px 1fr;
  gap: 6px;
  align-items: start;
}

.v-y-axis {
  position: relative;
  height: 252px;
  color: #6f7f83;
  font-size: 0.92rem;
}

.v-y-axis span {
  position: absolute;
  right: 0;
  transform: translateY(-50%);
  line-height: 1;
}

.v-y-axis span:nth-child(1) {
  top: 20%;
}
.v-y-axis span:nth-child(2) {
  top: 40%;
}
.v-y-axis span:nth-child(3) {
  top: 60%;
}
.v-y-axis span:nth-child(4) {
  top: 80%;
}
.v-y-axis span:nth-child(5) {
  top: 100%;
}

.v-chart-panel {
  display: flex;
  flex-direction: column;
}

.v-chart {
  height: 252px;
  position: relative;
  display: grid;
  grid-template-columns: repeat(7, minmax(92px, 1fr));
  align-items: end;
  gap: 8px;
  padding: 25px 8px 0 0;
  border-bottom: 1px solid #c9d2cc;
}

.v-grid-line {
  position: absolute;
  left: 0;
  right: 0;
  border-top: 1px solid #e3e8e5;
  z-index: 0;
}

.v-grid-line:nth-child(1) {
  top: 20%;
}
.v-grid-line:nth-child(2) {
  top: 40%;
}
.v-grid-line:nth-child(3) {
  top: 60%;
}
.v-grid-line:nth-child(4) {
  top: 80%;
}
.v-grid-line:nth-child(5) {
  top: 100%;
}

.v-col {
  position: relative;
  z-index: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
}

.v-col::before {
  content: '';
  position: absolute;
  top: 0;
  bottom: 0;
  left: -8px;
  right: -8px;
  border-radius: 2px;
  background: transparent;
  transition: background-color 0.15s ease;
  z-index: 0;
}

.v-col.is-active::before {
  background: #eef1f5;
}

.v-track {
  height: 100%;
  width: 100%;
  display: flex;
  align-items: end;
  justify-content: center;
  z-index: 1;
}

.v-bar {
  width: 92px;
  max-width: 100%;
  min-height: 8px;
  border-radius: 5px 5px 0 0;
  background: linear-gradient(180deg, #5dae83, var(--green-strong));
  position: relative;
  cursor: default;
  z-index: 2;
}

.v-bar-value {
  position: absolute;
  top: 8px;
  left: 50%;
  transform: translateX(-50%);
  color: #f4fbff;
  font-size: 0.95rem;
  font-weight: 700;
}

.v-x-axis {
  display: grid;
  grid-template-columns: repeat(7, minmax(92px, 1fr));
  gap: 8px;
  padding: 8px 8px 0 0;
}

.v-age {
  text-align: center;
  font-size: 1rem;
  color: #61727c;
}

.v-inline-tooltip {
  position: absolute;
  left: 50%;
  top: 53%;
  transform: translate(-30%, -50%);
  background: #ffffff;
  border: 1px solid #d9dde3;
  border-radius: 6px;
  box-shadow: 0 3px 8px rgba(34, 50, 63, 0.18);
  padding: 8px 10px;
  min-width: 132px;
  z-index: 4;
  pointer-events: none;
}

.v-tooltip-title {
  margin: 0;
  font-size: 0.85rem;
  color: #525f6c;
}

.v-tooltip-line {
  margin: 4px 0 0;
  display: flex;
  align-items: center;
  gap: 6px;
  color: #3e4f60;
  font-size: 0.95rem;
}

.v-tooltip-line strong {
  margin-left: auto;
  font-size: 1.2rem;
  line-height: 1;
}

.v-tooltip-dot {
  width: 10px;
  height: 10px;
  border-radius: 999px;
  background: var(--green-strong);
}

.two-col {
  margin-top: 16px;
  display: grid;
  gap: 16px;
  grid-template-columns: 1fr;
}

.card-title {
  margin: 0 0 12px;
  line-height: 1.35;
  font-size: 1.85rem;
  font-weight: 700;
}

.two-col .card {
  padding: 20px;
}

.mental-chart-card .card-title,
.local-wellbeing-card .card-title {
  font-size: 1.05rem;
  line-height: 1.35;
  margin-bottom: 10px;
}

.mini-title {
  margin: 34px 0 0;
  font-size: 0.95rem;
  color: #31484f;
}

.group-chart-wrap {
  margin-top: 30px;
  display: grid;
  grid-template-columns: 42px 1fr;
  gap: 8px;
  align-items: start;
}

.group-y-axis {
  position: relative;
  color: #6d7d82;
  font-size: 0.92rem;
}

.group-y-axis span {
  position: absolute;
  right: 0;
  transform: translateY(-50%);
  line-height: 1;
}

.group-y-axis--20 {
  height: 180px;
}

.group-y-axis--20 span:nth-child(1) {
  top: 0%;
}
.group-y-axis--20 span:nth-child(2) {
  top: 25%;
}
.group-y-axis--20 span:nth-child(3) {
  top: 50%;
}
.group-y-axis--20 span:nth-child(4) {
  top: 75%;
}
.group-y-axis--20 span:nth-child(5) {
  top: 100%;
}

.group-y-axis--25 {
  height: 180px;
}

.group-y-axis--25 span:nth-child(1) {
  top: 0%;
}
.group-y-axis--25 span:nth-child(2) {
  top: 20%;
}
.group-y-axis--25 span:nth-child(3) {
  top: 40%;
}
.group-y-axis--25 span:nth-child(4) {
  top: 60%;
}
.group-y-axis--25 span:nth-child(5) {
  top: 80%;
}
.group-y-axis--25 span:nth-child(6) {
  top: 100%;
}

.group-chart-panel {
  display: flex;
  flex-direction: column;
}

.group-chart {
  display: grid;
  grid-template-columns: repeat(3, minmax(90px, 1fr));
  position: relative;
  align-items: end;
  gap: 18px;
  padding: 0 0 0 2px;
  border-bottom: 1px solid #cfdad2;
}

.group-chart--20 {
  height: 180px;
}

.group-chart--25 {
  height: 180px;
}

.group-grid-line {
  position: absolute;
  left: 0;
  right: 0;
  border-top: 1px solid #e6ece8;
  z-index: 0;
}

.group-chart--20 .group-grid-line:nth-child(1) {
  top: 0%;
}
.group-chart--20 .group-grid-line:nth-child(2) {
  top: 25%;
}
.group-chart--20 .group-grid-line:nth-child(3) {
  top: 50%;
}
.group-chart--20 .group-grid-line:nth-child(4) {
  top: 75%;
}
.group-chart--20 .group-grid-line:nth-child(5) {
  top: 100%;
}

.group-chart--25 .group-grid-line:nth-child(1) {
  top: 0%;
}
.group-chart--25 .group-grid-line:nth-child(2) {
  top: 20%;
}
.group-chart--25 .group-grid-line:nth-child(3) {
  top: 40%;
}
.group-chart--25 .group-grid-line:nth-child(4) {
  top: 60%;
}
.group-chart--25 .group-grid-line:nth-child(5) {
  top: 80%;
}
.group-chart--25 .group-grid-line:nth-child(6) {
  top: 100%;
}

.group-x-axis {
  margin-top: 14px;
  display: grid;
  grid-template-columns: repeat(3, minmax(90px, 1fr));
  gap: 18px;
}

.group-col {
  position: relative;
  z-index: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
}

.group-col::before {
  content: '';
  position: absolute;
  top: 0;
  bottom: 0;
  left: -12px;
  right: -12px;
  border-radius: 4px;
  background: transparent;
  transition: background-color 0.2s ease;
  pointer-events: none;
}

.group-col.is-active::before {
  background: #edf4ee;
}

.pair-track {
  height: 100%;
  width: 100%;
  display: flex;
  align-items: end;
  justify-content: center;
  gap: 8px;
}

.pair-bar {
  width: 44px;
  min-height: 6px;
  border-radius: 4px 4px 0 0;
  cursor: pointer;
  position: relative;
}

.pair-bar--male {
  background: linear-gradient(180deg, #74b39e, var(--male));
}

.pair-bar--female {
  background: linear-gradient(180deg, #9fc27c, var(--female));
}

.group-inline-tooltip {
  position: absolute;
  top: 18px;
  left: 50%;
  transform: translateX(-50%);
  min-width: 166px;
  padding: 12px 14px;
  background: #ffffff;
  border: 1px solid #d6e4db;
  border-radius: 10px;
  box-shadow: 0 8px 18px rgba(27, 53, 41, 0.18);
  pointer-events: none;
  z-index: 5;
}

.group-tooltip-title {
  margin: 0 0 8px;
  font-size: 1.05rem;
  font-weight: 600;
  color: #4e6661;
}

.group-tooltip-line {
  margin: 0;
  display: grid;
  grid-template-columns: auto 1fr auto;
  align-items: center;
  gap: 8px;
  font-size: 1.02rem;
  color: #5a6f6a;
}

.group-tooltip-line + .group-tooltip-line {
  margin-top: 4px;
}

.group-tooltip-line strong {
  color: #3f5550;
}

.group-tooltip-dot {
  width: 12px;
  height: 12px;
  border-radius: 999px;
}

.group-tooltip-dot--male {
  background: var(--male);
}

.group-tooltip-dot--female {
  background: var(--female);
}

.pair-value {
  position: absolute;
  top: -26px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 0.86rem;
  font-weight: 600;
  color: #4b5e65;
  line-height: 1;
  white-space: nowrap;
}

.group-label {
  text-align: center;
  font-size: 0.98rem;
  color: #62747a;
}

.legend {
  margin-top: 18px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 22px;
  color: #55686e;
  font-size: 0.92rem;
}

.legend--chart {
  margin: 18px 0 30px;
}

.legend span {
  display: inline-flex;
  align-items: center;
}

.legend-dot {
  width: 12px;
  height: 12px;
  border-radius: 999px;
  display: inline-block;
  margin-right: 6px;
  vertical-align: middle;
}

.legend-dot.male {
  background: var(--male);
}
.legend-dot.female {
  background: var(--female);
}

.indicator-box {
  background: var(--kpi-bg);
  border: 1px solid #d2e9d9;
  border-radius: 10px;
  padding: 20px 18px;
  margin-top: 14px;
  text-align: center;
}

.indicator-value {
  margin: 0;
  font-size: clamp(2rem, 3.4vw, 3rem);
  line-height: 1.1;
  font-weight: 800;
  color: #1d8c54;
}

.indicator-label {
  margin: 8px 0 0;
  color: #4d6468;
  line-height: 1.7;
}

.mental-chart-card .source-text {
  margin-top: 14px;
  padding-top: 0;
}

.local-wellbeing-card .chart-note {
  margin-top: 22px;
  line-height: 1.9;
}

.local-wellbeing-card .source-text {
  margin-top: 16px;
  line-height: 1.75;
}

.local-wellbeing-card {
  display: flex;
  flex-direction: column;
}

.local-wellbeing-card .card-title {
  line-height: 1.5;
  margin-bottom: 14px;
}

.local-wellbeing-card .indicator-box {
  margin-top: 18px;
}

.local-wellbeing-card .indicator-label {
  line-height: 1.9;
}

.local-wellbeing-card .source-text {
  margin-top: 14px;
  padding-top: 0;
}

.section--impact {
  margin-top: 56px;
}

.impact-grid {
  margin-top: 14px;
  display: grid;
  grid-template-columns: 1fr;
  gap: 14px;
}

.impact-card {
  text-align: center;
  padding: 20px;
}

.impact-icon {
  font-size: 1.8rem;
  color: #2f6f7f;
}

.impact-value {
  margin: 10px 0 2px;
  font-size: 2.25rem;
  font-weight: 800;
  color: #2d6f85;
}

.impact-title {
  margin: 0;
  font-size: 1.25rem;
}

.impact-desc {
  margin: 10px 0 12px;
  color: #5f7275;
  line-height: 1.55;
}

.isolation-layout {
  margin-top: 14px;
  padding: 18px 20px;
  display: grid;
  gap: 18px;
  grid-template-columns: 1fr;
  align-items: start;
}

.community-cta-card {
  margin-top: 14px;
  padding: 30px 24px 28px;
  display: grid;
  justify-items: center;
  gap: 38px;
  text-align: center;
  border: 0;
  box-shadow: none;
  background: #ffffff;
}

.community-cta-highlight {
  margin: 0;
  width: auto;
  max-width: 100%;
  display: inline-block;
  border-radius: 12px;
  background: #e8f4ec;
  padding: 16px 24px;
  color: #3f7c5f;
  font-size: 2.16rem;
  font-weight: 600;
  line-height: 2;
  text-align: center;
}

.community-cta-button {
  border: 0;
  border-radius: 999px;
  padding: 14px 24px;
  min-width: min(100%, 420px);
  background: #76b792;
  color: #ffffff;
  font-family: inherit;
  font-size: 1.02rem;
  font-weight: 700;
  line-height: 1.35;
  cursor: pointer;
  box-shadow: 0 6px 14px rgba(22, 52, 68, 0.2);
  transition: transform 0.16s ease, box-shadow 0.16s ease, background-color 0.16s ease;
}

.community-cta-button:hover {
  background: #66ac85;
  transform: translateY(-1px);
  box-shadow: 0 10px 18px rgba(22, 52, 68, 0.24);
}

.community-cta-button:focus-visible {
  outline: 3px solid #8bb4ca;
  outline-offset: 2px;
}

.community-cta-desc {
  margin: 0;
  max-width: 760px;
  color: #101827;
  font-size: 0.92rem;
  line-height: 2.15;
}

.isolation-left {
  max-width: 620px;
  align-self: center;
  margin-left: 25px;
}

.isolation-title {
  margin: 0;
  font-size: 1.85rem;
  font-weight: 700;
  line-height: 1.25;
  color: #101827;
}

.isolation-intro {
  margin: 14px 0 0;
  max-width: 560px;
  color: #475467;
  font-size: 0.95rem;
  line-height: 1.65;
}

.isolation-fact-list {
  margin-top: 26px;
  display: grid;
  gap: 14px;
}

.isolation-fact-item {
  display: flex;
  align-items: flex-start;
  gap: 14px;
}

.isolation-fact-item + .isolation-fact-item {
  margin-top: 10px;
}

.fact-icon {
  flex: 0 0 26px;
  width: 26px;
  height: 26px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  margin-top: 4px;
  font-size: 1.35rem;
  font-weight: 700;
  line-height: 1;
}

.fact-icon-svg {
  width: 100%;
  height: 100%;
}

.fact-icon--warning {
  color: #f97316;
}

.fact-icon--health {
  color: #1d4f74;
}

.isolation-fact-item h4 {
  margin: 0;
  font-size: 1.8rem;
  font-weight: 700;
  color: #101827;
}

.isolation-fact-item p {
  margin: 8px 0 0;
  font-size: 0.95rem;
  line-height: 1.65;
  color: #475467;
}

.isolation-right {
  display: flex;
  justify-content: center;
  align-items: flex-start;
}

.orbit-visual {
  position: relative;
  width: min(100%, 440px);
  aspect-ratio: 1 / 1;
  min-height: 260px;
}

.orbit-ring {
  position: absolute;
  inset: 50%;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  border: 1px solid #dce3ec;
}

.orbit-ring--outer {
  width: 90%;
  height: 90%;
  border-color: #d4ddea;
  animation: orbitPulse 2.2s ease-in-out infinite;
}

.orbit-ring--middle {
  width: 66%;
  height: 66%;
  border-color: #d9e1eb;
}

.orbit-ring--inner {
  width: 42%;
  height: 42%;
  border-color: #dfe6ef;
}

.orbit-core {
  position: absolute;
  inset: 50%;
  transform: translate(-50%, -50%);
  width: 92px;
  height: 92px;
  border-radius: 50%;
  border: 2px solid #cfd9e3;
  background: #ffffff;
  box-shadow: 0 6px 16px rgba(16, 24, 40, 0.08);
  display: flex;
  align-items: center;
  justify-content: center;
}

.home-icon {
  width: 42px;
  height: 42px;
  border-radius: 10px;
  background: #5dae83;
  color: #ffffff;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 1.55rem;
}

.orbit-label {
  position: absolute;
  white-space: nowrap;
  border: 1px solid #d9dfeb;
  background: #ffffff;
  color: #344054;
  border-radius: 999px;
  padding: 7px 17px;
  font-size: 0.98rem;
  font-weight: 600;
  box-shadow: 0 2px 6px rgba(16, 24, 40, 0.08);
  transition:
    background-color 0.2s ease,
    border-color 0.2s ease,
    color 0.2s ease,
    box-shadow 0.2s ease;
}

.orbit-label:hover {
  background: #e8f4ec;
  border-color: #a7ceb6;
  color: #24543f;
  box-shadow: 0 6px 14px rgba(35, 92, 66, 0.18);
}

.orbit-label--top {
  top: 2%;
  left: 50%;
  transform: translateX(-50%);
}

.orbit-label--top-right {
  top: 22%;
  right: 0;
}

.orbit-label--bottom-right {
  bottom: 24%;
  right: 1%;
}

.orbit-label--bottom {
  bottom: 2%;
  left: 50%;
  transform: translateX(-50%);
}

.orbit-label--bottom-left {
  bottom: 24%;
  left: 1%;
}

.orbit-label--top-left {
  top: 22%;
  left: 0;
}

@keyframes orbitPulse {
  0%,
  100% {
    opacity: 0.42;
    box-shadow: 0 0 0 0 rgba(134, 154, 189, 0.2);
  }
  50% {
    opacity: 1;
    box-shadow: 0 0 0 8px rgba(134, 154, 189, 0.14);
  }
}

.source-text {
  margin: 10px 0 0;
  font-size: 0.82rem;
  color: #7a8d91;
}

.chart-note {
  margin: 12px 0 0;
  line-height: 1.62;
  color: #5f7275;
}

.chart-tooltip {
  position: fixed;
  z-index: 50;
  pointer-events: none;
  background: rgba(34, 51, 44, 0.94);
  color: #f4faf6;
  border: 1px solid rgba(207, 233, 215, 0.5);
  border-radius: 8px;
  padding: 6px 10px;
  font-size: 0.8rem;
  box-shadow: 0 8px 18px rgba(0, 0, 0, 0.15);
  white-space: nowrap;
}

@media (min-width: 900px) {
  .issue-card {
    grid-template-columns: 1.2fr 1fr;
    align-items: stretch;
  }

  .two-col {
    grid-template-columns: 1fr 1fr;
    align-items: start;
  }

  .impact-grid {
    grid-template-columns: repeat(3, 1fr);
  }

  .isolation-layout {
    grid-template-columns: minmax(0, 1fr) minmax(320px, 0.78fr);
    align-items: start;
  }
}

@media (max-width: 720px) {
  .v-chart-wrap {
    grid-template-columns: 44px 1fr;
    gap: 6px;
  }

  .v-y-axis {
    height: 214px;
    font-size: 0.78rem;
  }

  .v-chart {
    grid-template-columns: repeat(7, minmax(42px, 1fr));
    gap: 10px;
    padding: 20px 0 0;
    height: 214px;
  }

  .v-x-axis {
    grid-template-columns: repeat(7, minmax(42px, 1fr));
    gap: 10px;
    padding-right: 0;
  }

  .v-bar {
    width: 100%;
  }

  .v-age {
    font-size: 0.82rem;
  }

  .donut-item {
    min-width: 100%;
  }

  .isolation-layout {
    padding: 14px;
    gap: 14px;
  }

  .community-cta-card {
    padding: 20px 14px 18px;
    gap: 26px;
  }

  .community-cta-highlight {
    font-size: 1.5rem;
    line-height: 1.95;
    padding: 14px 16px;
  }

  .community-cta-button {
    width: 100%;
    min-width: 0;
    padding: 12px 12px;
    font-size: 0.96rem;
  }

  .community-cta-desc {
    margin-top: 0;
    font-size: 0.88rem;
    line-height: 2.05;
  }

  .isolation-title {
    font-size: 1.55rem;
  }

  .isolation-intro {
    font-size: 0.92rem;
    margin-top: 14px;
  }

  .isolation-fact-item h4 {
    font-size: 1.25rem;
  }

  .isolation-fact-item p {
    font-size: 0.9rem;
  }

  .orbit-visual {
    width: min(100%, 440px);
    min-height: 280px;
  }

  .orbit-label {
    font-size: 0.86rem;
    padding: 6px 12px;
  }

  .orbit-label--top-right {
    right: -2%;
  }

  .orbit-label--bottom-right {
    right: -2%;
  }

  .orbit-label--bottom-left {
    left: -2%;
  }

  .orbit-label--top-left {
    left: -2%;
  }

  .orbit-core {
    width: 78px;
    height: 78px;
  }

  .home-icon {
    width: 36px;
    height: 36px;
    font-size: 1.25rem;
  }
}
</style>
