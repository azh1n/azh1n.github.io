<!-- Pattern view component for displaying and interacting with knitting patterns -->
<template>
  <!-- Main pattern view container -->
  <div class="pattern-view">
    <!-- Pattern header with title, controls, and progress -->
    <div class="pattern-header">
      <div class="header-content">
        <h1>{{ pattern.name }}</h1>
        <div class="pattern-controls">
          <button @click="confirmDelete" class="delete-button">
            <font-awesome-icon icon="trash" />
            Delete Pattern
          </button>
        </div>
      </div>
      <!-- Progress bar showing completion status -->
      <div class="progress-bar">
        <div class="progress-track">
          <div 
            class="progress-fill"
            :style="{ width: `${completionPercentage}%` }"
          ></div>
        </div>
        <span class="progress-text">{{ completedRows }} of {{ totalRows }} rows completed</span>
      </div>
    </div>

    <!-- Main pattern content area -->
    <div class="pattern-content">
      <!-- Row header with current row info and controls -->
      <div class="row-header">
        <div class="row-info">
          <div class="row-title">
            <h2>Row {{ currentRow?.rowNum }}</h2>
            <span class="row-color">{{ currentRow?.color }}</span>
          </div>
          <button 
            @click="toggleRowComplete"
            :class="['complete-button', { 'completed': isRowComplete }]"
          >
            <font-awesome-icon :icon="isRowComplete ? 'check-circle' : 'circle'" />
            {{ isRowComplete ? 'Completed' : 'Mark Complete' }}
          </button>
        </div>
        
        <!-- Stitches per view control -->
        <div class="stitch-control">
          <label for="stitchesPerView">Stitches per view:</label>
          <div class="number-control">
            <button 
              @click="decreaseStitches" 
              class="control-button"
              :disabled="stitchesPerView <= 1"
            >
              −
            </button>
            <input 
              type="number" 
              id="stitchesPerView" 
              v-model="stitchesPerView" 
              min="1"
              :max="totalStitches"
              class="number-input"
            />
            <button 
              @click="increaseStitches" 
              class="control-button"
              :disabled="stitchesPerView >= totalStitches"
            >
              +
            </button>
          </div>
        </div>
      </div>

      <!-- Pattern card with stitch navigation -->
      <div class="pattern-card">
        <div class="stitch-navigation">
          <button 
            @click="previousStitches" 
            class="nav-button"
            :disabled="currentStitchIndex === 0"
          >
            <font-awesome-icon icon="chevron-left" />
          </button>
          
          <!-- Current stitches display -->
          <div class="stitch-content">
            <div class="current-stitches">
              <span 
                v-for="(stitch, index) in currentStitches" 
                :key="index"
                :class="{ 'completed-stitch': stitch.isCompleted }"
              >
                {{ stitch.code }}
              </span>
            </div>
            <div class="stitch-progress">
              <span class="progress-indicator">
                {{ stitchProgress }}
              </span>
            </div>
          </div>
          
          <button 
            @click="nextStitches" 
            class="nav-button"
            :disabled="currentStitchIndex + stitchesPerView >= totalStitches"
          >
            <font-awesome-icon icon="chevron-right" />
          </button>
        </div>

        <!-- Full row preview section -->
        <div class="full-row-preview">
          <h3>Full Row Preview</h3>
          <div class="preview-content">
            <template v-if="windowWidth < 768">
              <span 
                v-for="(item, index) in mobilePreviewStitches" 
                :key="index"
                :class="[
                  'preview-stitch',
                  { 'completed-stitch': item.status === 'completed' },
                  { 'current-stitch': item.status === 'current' },
                  { 'next-stitch': item.status === 'next' },
                  { 'border-stitch': item.code.endsWith('bs') }
                ]"
              >
                {{ item.code }}
              </span>
            </template>
            <template v-else>
              <span 
                v-for="(item, index) in getCompletedCodes" 
                :key="index"
                :class="[
                  'preview-stitch',
                  { 'completed-stitch': item.isCompleted },
                  { 'current-stitch': index >= currentStitchIndex && index < currentStitchIndex + stitchesPerView },
                  { 'border-stitch': item.code.endsWith('bs') }
                ]"
              >
                {{ item.code }}
              </span>
            </template>
          </div>
        </div>
      </div>

      <!-- Row navigation controls -->
      <div class="row-navigation">
        <button 
          @click="previousRow" 
          class="nav-button large"
          :disabled="currentRowIndex === 0"
        >
          <font-awesome-icon icon="arrow-left" />
          <span>Previous Row</span>
        </button>
        <div class="row-selector">
          <span class="row-counter">Row</span>
          <select 
            v-model="currentRowIndex" 
            class="row-select"
          >
            <option 
              v-for="(row, index) in parsedRows" 
              :key="index" 
              :value="index"
            >
              {{ row.rowNum }} ({{ row.color }})
            </option>
          </select>
          <span class="row-counter">of {{ parsedRows.length }}</span>
        </div>
        <button 
          @click="nextRow" 
          class="nav-button large"
          :disabled="currentRowIndex === parsedRows.length - 1"
        >
          <font-awesome-icon icon="arrow-right" />
          <span>Next Row</span>
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { updateDoc, doc, deleteDoc } from 'firebase/firestore'
import { db } from '../firebase'

// Component props
const props = defineProps({
  pattern: {
    type: Object,
    required: true  // Pattern object containing name, content, and completion data
  },
  patterns: {
    type: Array,
    required: true  // Array of all patterns
  },
  currentTextIndex: {
    type: Number,
    required: true  // Current text index for navigation
  }
})

// Event emitters and router
const emit = defineEmits(['update:currentTextIndex', 'pattern-deleted'])
const router = useRouter()

// Reactive state
const stitchesPerView = ref(5)  // Number of stitches to display at once
const currentStitchIndex = ref(0)  // Current stitch index in the row
const currentRowIndex = ref(0)  // Current row index in the pattern
const isSwiping = ref(false)  // Touch swipe state
const startX = ref(0)  // Touch start position
const currentX = ref(0)  // Current touch position
const windowWidth = ref(window.innerWidth)  // Current window width

// Parse pattern rows into structured data
const parsedRows = computed(() => {
  if (!props.pattern) return []
  
  const content = props.pattern.content
  const rows = content.split('\n').filter(row => row.trim())
  const parsedRows = []
  
  let currentRow = []
  let currentRowNum = null
  let currentColor = null
  
  // Process pattern codes into structured format
  const processPattern = (pattern) => {
    const parts = pattern.split(/,(?![^(]*\))/).map(p => p.trim())
    return parts.filter(code => 
      code.match(/^\d+[a-z]+$/) || 
      code.match(/^\([^)]+\)x\d+$/)
    )
  }
  
  // Parse each row of the pattern
  rows.forEach(row => {
    const rowMatch = row.match(/Row (\d+): With (Color [A-Z])/)
    if (rowMatch) {
      if (currentRow.length > 0) {
        parsedRows.push({
          rowNum: currentRowNum,
          color: currentColor,
          codes: currentRow,
          fullRow: currentRow.join(' ')
        })
      }
      
      currentRowNum = rowMatch[1]
      currentColor = rowMatch[2]
      currentRow = []
      
      let pattern = row.split(currentColor)[1]
      const codes = processPattern(pattern)
      currentRow.push(...codes)
    } else {
      let pattern = row
      if (pattern.includes('FO.')) {
        pattern = pattern.split('FO.')[0]
      }
      if (pattern.includes('(Stitch Count')) {
        pattern = pattern.split('(Stitch Count')[0]
      }
      
      const codes = processPattern(pattern)
      currentRow.push(...codes)
    }
  })
  
  if (currentRow.length > 0) {
    parsedRows.push({
      rowNum: currentRowNum,
      color: currentColor,
      codes: currentRow,
      fullRow: currentRow.join(' ')
    })
  }
  
  return parsedRows
})

// Computed properties for current state
const currentRow = computed(() => {
  return parsedRows.value[currentRowIndex.value]
})

const visibleStitches = computed(() => {
  if (!currentRow.value) return ''
  const start = currentStitchIndex.value
  const end = Math.min(start + stitchesPerView.value, currentRow.value.codes.length)
  return currentRow.value.codes.slice(start, end).join(' ')
})

// Calculate total stitches in current row
const totalStitches = computed(() => {
  if (!currentRow.value) return 0
  return currentRow.value.codes.length
})

// Get completed codes for full row preview
const getCompletedCodes = computed(() => {
  if (!currentRow.value) return []
  const completedIndex = currentStitchIndex.value + stitchesPerView.value
  return currentRow.value.codes.map((code, index) => ({
    code,
    isCompleted: index < completedIndex
  }))
})

// Check if current row is marked as complete
const isRowComplete = computed(() => {
  if (!currentRow.value || !props.pattern?.completedRows) return false
  return props.pattern.completedRows[`row${currentRow.value.rowNum}`] || false
})

// Touch swipe transform style
const transformStyle = computed(() => {
  if (!isSwiping.value) return ''
  const diff = currentX.value - startX.value
  return `translateX(${diff}px)`
})

// Navigation methods
const nextStitches = () => {
  if (!currentRow.value) return
  let currentTotal = 0
  let targetIndex = currentStitchIndex.value

  // Calculate the total stitches up to the current index
  for (let i = 0; i < currentStitchIndex.value; i++) {
    const code = currentRow.value.codes[i]
    const count = parseInt(code.match(/\d+/)[0], 10)
    currentTotal += count
  }

  // Find the next index that would add approximately stitchesPerView.value stitches
  while (targetIndex < currentRow.value.codes.length) {
    const code = currentRow.value.codes[targetIndex]
    const count = parseInt(code.match(/\d+/)[0], 10)
    if (currentTotal + count > stitchesPerView.value) break
    currentTotal += count
    targetIndex++
  }

  if (targetIndex > currentStitchIndex.value) {
    currentStitchIndex.value = targetIndex
    updateScrollPosition()
  }
}

const previousStitches = () => {
  if (!currentRow.value || currentStitchIndex.value === 0) return
  let currentTotal = 0
  let targetIndex = currentStitchIndex.value - 1

  // Calculate backwards until we find the right starting point
  while (targetIndex > 0) {
    const code = currentRow.value.codes[targetIndex]
    const count = parseInt(code.match(/\d+/)[0], 10)
    if (currentTotal + count > stitchesPerView.value) break
    currentTotal += count
    targetIndex--
  }

  currentStitchIndex.value = targetIndex
  updateScrollPosition()
}

// Update stitch progress display
const stitchProgress = computed(() => {
  if (!currentRow.value) return ''
  const start = currentStitchIndex.value + 1
  const end = Math.min(currentStitchIndex.value + stitchesPerView.value, totalStitches.value)
  return `${start}-${end} of ${totalStitches.value} stitches`
})

// Row navigation with completion tracking
const nextRow = async () => {
  if (currentRowIndex.value < parsedRows.value.length - 1) {
    // Mark current row as complete
    if (currentRow.value && !isRowComplete.value) {
      try {
        const textId = props.pattern.id
        const completionData = props.pattern.completedRows || {}
        completionData[`row${currentRow.value.rowNum}`] = true
        
        await updateDoc(doc(db, 'patterns', textId), {
          completedRows: completionData
        })
      } catch (error) {
        console.error('Error updating row completion:', error)
      }
    }
    
    // Move to next row
    currentRowIndex.value++
    currentStitchIndex.value = 0
  }
}

const previousRow = () => {
  if (currentRowIndex.value > 0) {
    currentRowIndex.value--
    currentStitchIndex.value = 0
  }
}

// Toggle row completion status
const toggleRowComplete = async () => {
  if (!currentRow.value) return
  
  try {
    const textId = props.pattern.id
    const completionData = props.pattern.completedRows || {}
    completionData[`row${currentRow.value.rowNum}`] = !completionData[`row${currentRow.value.rowNum}`]
    
    await updateDoc(doc(db, 'patterns', textId), {
      completedRows: completionData
    })
  } catch (error) {
    console.error('Error updating row completion:', error)
  }
}

// Touch event handlers
const handleTouchStart = (e) => {
  isSwiping.value = true
  startX.value = e.touches[0].clientX
  currentX.value = startX.value
}

const handleTouchMove = (e) => {
  if (!isSwiping.value) return
  currentX.value = e.touches[0].clientX
}

const handleTouchEnd = () => {
  if (!isSwiping.value) return
  
  const diff = startX.value - currentX.value
  if (Math.abs(diff) > 100) {
    if (diff > 0) {
      nextRow()
    } else {
      previousRow()
    }
  }
  
  isSwiping.value = false
  startX.value = 0
  currentX.value = 0
}

// Pattern deletion
const confirmDelete = () => {
  if (confirm('Are you sure you want to delete this pattern? This action cannot be undone.')) {
    deletePattern()
  }
}

const deletePattern = async () => {
  try {
    await deleteDoc(doc(db, 'patterns', props.pattern.id))
    emit('pattern-deleted')
    router.push('/')
  } catch (error) {
    console.error('Error deleting pattern:', error)
    alert('Failed to delete pattern. Please try again.')
  }
}

// Completion tracking
const completedRows = computed(() => {
  if (!props.pattern?.completedRows) return 0
  return Object.values(props.pattern.completedRows).filter(Boolean).length
})

const totalRows = computed(() => parsedRows.value.length)

const completionPercentage = computed(() => {
  if (totalRows.value === 0) return 0
  return Math.round((completedRows.value / totalRows.value) * 100)
})

// Current stitches display
const currentStitches = computed(() => {
  if (!currentRow.value) return []
  const start = currentStitchIndex.value
  const end = start + stitchesPerView.value
  return currentRow.value.codes.slice(start, end).map((code, index) => ({
    code,
    isCompleted: index < stitchesPerView.value
  }))
})

// Stitch view controls
const decreaseStitches = () => {
  if (stitchesPerView.value > 1) {
    stitchesPerView.value--
  }
}

const increaseStitches = () => {
  if (stitchesPerView.value < totalStitches.value) {
    stitchesPerView.value++
  }
}

// Separate function for scroll position update
const updateScrollPosition = () => {
  // Wait for DOM to update
  setTimeout(() => {
    const previewContent = document.querySelector('.preview-content')
    if (!previewContent || !currentRow.value) return

    // Center the current stitches
    const currentStitches = previewContent.querySelectorAll('.current-stitch')
    if (currentStitches.length === 0) return

    const firstCurrentStitch = currentStitches[0]
    const containerWidth = previewContent.offsetWidth
    const scrollPosition = firstCurrentStitch.offsetLeft - (containerWidth / 2) + (firstCurrentStitch.offsetWidth * stitchesPerView.value / 2)
    
    previewContent.scrollTo({
      left: Math.max(0, scrollPosition),
      behavior: 'smooth'
    })
  }, 0)
}

// Watch for window resize
onMounted(() => {
  window.addEventListener('resize', () => {
    windowWidth.value = window.innerWidth
  })
})

// New computed property for mobile preview stitches
const mobilePreviewStitches = computed(() => {
  if (!currentRow.value) return []
  
  const allStitches = currentRow.value.codes
  const result = []
  
  // Add all completed stitches
  for (let i = 0; i < currentStitchIndex.value; i++) {
    result.push({ code: allStitches[i], status: 'completed' })
  }
  
  // Add current visible stitches
  for (let i = currentStitchIndex.value; i < currentStitchIndex.value + stitchesPerView.value; i++) {
    if (i < allStitches.length) {
      result.push({ code: allStitches[i], status: 'current' })
    }
  }
  
  // Add next 3 incomplete stitches
  let nextCount = 0
  for (let i = currentStitchIndex.value + stitchesPerView.value; i < allStitches.length && nextCount < 3; i++) {
    result.push({ code: allStitches[i], status: 'next' })
    nextCount++
  }
  
  return result
})

// Component initialization
onMounted(() => {
  // Find the first incomplete row
  const firstIncompleteRowIndex = parsedRows.value.findIndex(row => {
    return !props.pattern?.completedRows?.[`row${row.rowNum}`]
  })
  
  // If an incomplete row is found, navigate to it
  if (firstIncompleteRowIndex !== -1) {
    currentRowIndex.value = firstIncompleteRowIndex
  }
})
</script>

<style scoped>
/* Main container styles */
.pattern-view {
  padding: 1rem;
  color: var(--text-primary);
}

/* Header section styles */
.pattern-header {
  margin-bottom: 2rem;
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.header-content h1 {
  margin: 0;
  font-size: 2rem;
  color: var(--text-primary);
}

/* Delete button styles */
.delete-button {
  padding: 0.8rem 1.5rem;
  border: 1px solid #ff4444;
  border-radius: 8px;
  background: transparent;
  color: #ff4444;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.delete-button:hover {
  background-color: rgba(255, 68, 68, 0.1);
}

/* Progress bar styles */
.progress-bar {
  margin-top: 1rem;
}

.progress-track {
  width: 100%;
  height: 4px;
  background-color: var(--border-color);
  border-radius: 2px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background-color: var(--accent-color);
  transition: width 0.3s ease;
}

.progress-text {
  display: block;
  margin-top: 0.5rem;
  color: var(--text-secondary);
  font-size: 0.9rem;
}

/* Pattern content styles */
.pattern-content {
  background-color: var(--card-bg);
  border-radius: 16px;
  border: 1px solid var(--border-color);
  overflow: hidden;
}

/* Row header styles */
.row-header {
  padding: 1.5rem;
  border-bottom: 1px solid var(--border-color);
}

.row-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.row-title {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.row-title h2 {
  margin: 0;
  font-size: 1.5rem;
  color: var(--text-primary);
}

.row-color {
  color: var(--accent-color);
  font-weight: 500;
}

/* Complete button styles */
.complete-button {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  border: 2px solid var(--accent-color);
  border-radius: 8px;
  background: transparent;
  color: var(--accent-color);
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.complete-button:hover {
  background-color: rgba(76, 175, 80, 0.1);
}

.complete-button.completed {
  background-color: var(--accent-color);
  color: white;
}

/* Stitch control styles */
.stitch-control {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-top: 1rem;
  color: var(--text-primary);
}

.number-control {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

/* Control button styles */
.control-button {
  padding: 0.5rem;
  width: 32px;
  height: 32px;
  border: 1px solid var(--button-border);
  border-radius: 6px;
  background: var(--button-bg);
  color: var(--button-text);
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.2rem;
}

.control-button:hover:not(:disabled) {
  background: var(--button-hover-bg);
  border-color: var(--accent-color);
}

.control-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Number input styles */
.number-input {
  width: 60px;
  padding: 0.5rem;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  background-color: var(--input-bg);
  color: var(--text-primary);
  text-align: center;
  font-size: 1rem;
  -moz-appearance: textfield;
}

.number-input:focus {
  outline: none;
  border-color: var(--accent-color);
  background-color: var(--hover-bg);
}

.number-input::-webkit-outer-spin-button,
.number-input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

/* Pattern card styles */
.pattern-card {
  padding: 2rem;
  background-color: var(--card-bg);
}

/* Stitch navigation styles */
.stitch-navigation {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 2rem;
}

/* Navigation button styles */
.nav-button {
  padding: 0.8rem;
  border: 1px solid var(--button-border);
  border-radius: 8px;
  background: var(--button-bg);
  color: var(--button-text);
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.nav-button:hover:not(:disabled) {
  background: var(--button-hover-bg);
  border-color: var(--accent-color);
}

.nav-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.nav-button.large {
  padding: 1rem 2rem;
  font-size: 1.1rem;
}

/* Stitch content styles */
.stitch-content {
  flex: 1;
  text-align: center;
}

.current-stitches {
  font-size: 1.8rem;
  margin-bottom: 1rem;
  display: flex;
  justify-content: center;
  gap: 1rem;
  color: var(--accent-color);
}

.stitch-progress {
  color: var(--text-secondary);
  font-size: 0.9rem;
}

/* Full row preview styles */
.full-row-preview {
  margin-top: 2rem;
  padding-top: 2rem;
  border-top: 1px solid var(--border-color);
}

.preview-content {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 1rem;
  overflow-x: auto;
  padding-bottom: 1rem;
  -webkit-overflow-scrolling: touch;
}

.preview-stitch {
  font-size: 1rem;
  color: var(--text-primary);
  position: relative;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
}

.preview-stitch.completed-stitch {
  color: var(--accent-color);
}

.preview-stitch.current-stitch {
  font-weight: bold;
  color: var(--text-primary);
  background-color: rgba(76, 175, 80, 0.1);
}

.preview-stitch.next-stitch {
  color: var(--text-secondary);
}

.preview-stitch.border-stitch {
  color: var(--accent-color);
  font-weight: 600;
  background-color: rgba(76, 175, 80, 0.1);
  border: 1px solid var(--accent-color);
}

@media (max-width: 767px) {
  .pattern-view {
    padding: 0.5rem;
  }

  .pattern-header {
    margin-bottom: 1rem;
  }

  .header-content {
    margin-bottom: 0.5rem;
  }

  .header-content h1 {
    font-size: 1.5rem;
  }

  .row-title h2 {
    font-size: 1.2rem;
  }

  .row-color {
    font-size: 1rem;
  }

  .row-header {
    padding: 0.75rem;
  }

  .complete-button {
    padding: 0.35rem 0.75rem;
    font-size: 0.9rem;
    gap: 0.35rem;
  }

  .stitch-control {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
    margin-top: 0.75rem;
  }

  .number-control {
    width: 100%;
    justify-content: space-between;
  }

  .number-input {
    width: 80px;
  }

  .pattern-card {
    padding: 0.75rem;
  }

  .stitch-navigation {
    margin-bottom: 1rem;
  }

  .nav-button {
    padding: 0.5rem;
  }

  .nav-button.large {
    width: 48px;
    height: 48px;
    padding: 0;
    justify-content: center;
  }

  .nav-button.large span {
    display: none;
  }

  .current-stitches {
    font-size: 1.5rem;
    flex-wrap: wrap;
    gap: 0.5rem;
  }

  .full-row-preview {
    margin-top: 1rem;
    padding-top: 1rem;
  }

  .preview-content {
    flex-wrap: nowrap;
    justify-content: flex-start;
    padding: 0.5rem;
    overflow-x: auto;
    -webkit-overflow-scrolling: touch;
    scroll-padding: 1rem;
    gap: 0.75rem;
    margin-top: 0.5rem;
  }
  
  .preview-stitch {
    flex-shrink: 0;
    font-size: 1.2rem;
    padding: 0.5rem;
    white-space: nowrap;
  }

  .row-navigation {
    padding: 0.75rem;
    flex-direction: row;
    gap: 0.5rem;
  }

  .row-selector {
    flex: 1;
    justify-content: center;
  }

  .row-select {
    max-width: none;
    width: 120px;
  }

  .stitch-progress {
    margin-top: 0.5rem;
    font-size: 0.8rem;
  }
}

/* Row navigation styles */
.row-navigation {
  padding: 1.5rem;
  border-top: 1px solid var(--border-color);
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 2rem;
}

/* Row selector styles */
.row-selector {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.row-counter {
  color: var(--text-secondary);
}

.row-select {
  padding: 0.5rem;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  background-color: var(--input-bg);
  color: var(--text-primary);
  font-size: 1rem;
  min-width: 150px;
}

.row-select:focus {
  outline: none;
  border-color: var(--accent-color);
  background-color: var(--hover-bg);
}

/* Responsive styles */
@media (min-width: 1024px) {
  .pattern-view {
    padding: 2rem;
  }

  .header-content h1 {
    font-size: 2.5rem;
  }

  .row-title h2 {
    font-size: 1.8rem;
  }

  .current-stitches {
    font-size: 2rem;
  }
}
</style> 