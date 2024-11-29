<template>
  <div class="container">
    <h1>Secret Santa Generator</h1>
    
    <!-- Toggle button for setup section -->
    <button 
      @click="isSetupVisible = !isSetupVisible"
      class="toggle-btn"
    >
      {{ isSetupVisible ? 'Hide Setup' : 'Show Setup' }}
    </button>

    <!-- Wrap the setup sections in a div with v-show -->
    <div v-show="isSetupVisible">
      <!-- Add Family Form -->
      <div class="form-section">
        <h2>Add Family</h2>
        <input 
          v-model="newFamilyName" 
          placeholder="Enter family name"
          @keyup.enter="addFamily"
        >
        <button @click="addFamily">Add Family</button>
      </div>

      <!-- Family Lists -->
      <div class="families-section">
        <div v-for="(family, familyIndex) in families" :key="familyIndex" class="family-card">
          <h3>{{ family.name }}</h3>
          
          <!-- Add Member Form -->
          <div class="member-form">
            <input 
              v-model="family.newMember" 
              placeholder="Add family member"
              @keyup.enter="addMember(familyIndex)"
            >
            <button @click="addMember(familyIndex)">Add</button>
          </div>
          
          <!-- Member List -->
          <ul>
            <li v-for="(member, memberIndex) in family.members" :key="memberIndex">
              {{ member }}
              <button 
                class="delete-btn"
                @click="removeMember(familyIndex, memberIndex)"
              >×</button>
            </li>
          </ul>
          
          <button 
            class="delete-family"
            @click="removeFamily(familyIndex)"
          >Delete Family</button>
        </div>
      </div>
    </div>

    <!-- Actions -->
    <div class="actions">
      <button 
        @click="assignSantas" 
        :disabled="!canAssign"
        class="primary-btn"
      >
        Assign Santas
      </button>
      <div class="reset-buttons">
        <button @click="resetAssignments" class="secondary-btn">
          Reset Assignments
        </button>
        <button @click="resetEverything" class="warning-btn">
          Reset Names
        </button>
      </div>
    </div>

    <!-- Reveal Section -->
    <div v-if="assignments.length" class="reveal-section">
      <h2>Reveal Assignments</h2>
      <div class="assignment-card" v-if="currentRevealIndex < assignments.length && assignments[currentRevealIndex]">
        <!-- Step 1: Initial handoff -->
        <div v-if="revealStep === 1" class="reveal-step">
          <h3>Give the device to</h3>
          <p class="name-display">{{ assignments[currentRevealIndex].santa }}</p>
          <button @click="nextStep" class="reveal-btn">
            I have the device
          </button>
        </div>

        <!-- Step 2: Confirm ready -->
        <div v-if="revealStep === 2" class="reveal-step">
          <h3>{{ assignments[currentRevealIndex].santa }}'s turn</h3>
          <p>Press reveal when you're ready to see your assignment</p>
          <button @click="nextStep" class="reveal-btn">
            Reveal My Assignment
          </button>
        </div>

        <!-- Step 3: Show assignment -->
        <div v-if="revealStep === 3" class="reveal-step">
          <h3>{{ assignments[currentRevealIndex].santa }},</h3>
          <p>You are Secret Santa for:</p>
          <p class="name-display">{{ assignments[currentRevealIndex].recipient }}</p>
          <button @click="nextStep" class="reveal-btn">
            Hide Answer
          </button>
        </div>

        <!-- Step 4: Transition to next person -->
        <div v-if="revealStep === 4" class="reveal-step">
          <h3>Now give the device to {{ assignments[currentRevealIndex + 1]?.santa || 'Finished!' }}</h3>
          <button @click="nextPerson" class="reveal-btn">
            {{ currentRevealIndex < assignments.length - 1 ? 'I have the device' : 'Finish' }}
          </button>
        </div>
      </div>
      <div v-else-if="currentRevealIndex === -1" class="reveal-step">
        <button @click="startReveal" class="reveal-btn">
          Start Revealing
        </button>
      </div>
      <p v-else>All assignments have been revealed!</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'

const families = ref(JSON.parse(localStorage.getItem('secretSantaFamilies')) || [])
const assignments = ref(JSON.parse(localStorage.getItem('secretSantaAssignments')) || [])
const currentRevealIndex = ref(-1)
const newFamilyName = ref('')
const revealStep = ref(1)
const isSetupVisible = ref(!assignments.value.length)

watch(families, (newFamilies) => {
  localStorage.setItem('secretSantaFamilies', JSON.stringify(newFamilies))
}, { deep: true })

watch(assignments, (newAssignments) => {
  localStorage.setItem('secretSantaAssignments', JSON.stringify(newAssignments))
}, { deep: true })

// Computed property to check if we can make assignments
const canAssign = computed(() => {
  const totalMembers = families.value.reduce((sum, family) => sum + family.members.length, 0)
  return totalMembers >= 2 && families.value.some(family => family.members.length > 0)
})

// Add a new family
function addFamily() {
  if (newFamilyName.value.trim()) {
    families.value.push({
      name: newFamilyName.value.trim(),
      members: [],
      newMember: ''
    })
    newFamilyName.value = ''
  }
}

// Add a member to a family
function addMember(familyIndex) {
  const family = families.value[familyIndex]
  if (family.newMember.trim()) {
    family.members.push(family.newMember.trim())
    family.newMember = ''
  }
}

// Remove a member from a family
function removeMember(familyIndex, memberIndex) {
  families.value[familyIndex].members.splice(memberIndex, 1)
}

// Remove a family
function removeFamily(familyIndex) {
  families.value.splice(familyIndex, 1)
}

// Reset everything
function resetAssignments() {
  assignments.value = []
  currentRevealIndex.value = -1
  revealStep.value = 1
}

function resetEverything() {
  if (confirm('Are you sure you want to delete all families and names?')) {
    families.value = []
    assignments.value = []
    currentRevealIndex.value = -1
    revealStep.value = 1
    newFamilyName.value = ''
    localStorage.removeItem('secretSantaFamilies')
    localStorage.removeItem('secretSantaAssignments')
  }
}

// Assign Secret Santas
function assignSantas() {
  // Create flat list of all members with their family index
  const allMembers = families.value.flatMap((family, familyIndex) =>
    family.members.map(member => ({ name: member, familyIndex }))
  )
  
  // Shuffle recipients
  const recipients = [...allMembers]
  let attempts = 0
  const maxAttempts = 100
  
  while (attempts < maxAttempts) {
    // Shuffle recipients
    for (let i = recipients.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [recipients[i], recipients[j]] = [recipients[j], recipients[i]]
    }
    
    // Check if assignment is valid
    let valid = true
    for (let i = 0; i < allMembers.length; i++) {
      if (
        allMembers[i].name === recipients[i].name || // Same person
        allMembers[i].familyIndex === recipients[i].familyIndex // Same family
      ) {
        valid = false
        break
      }
    }
    
    if (valid) {
      assignments.value = allMembers.map((santa, i) => ({
        santa: santa.name,
        recipient: recipients[i].name
      }))
      currentRevealIndex.value = -1
      revealStep.value = 1
      isSetupVisible.value = false  // Auto-hide setup when assignments are made
      return
    }
    
    attempts++
  }
  
  alert('Could not find valid assignments. Please try again.')
}

// Reveal next assignment
function nextStep() {
  if (revealStep.value === 4) {
    nextPerson()
  } else {
    revealStep.value++
  }
}

function nextPerson() {
  if (currentRevealIndex.value < assignments.value.length - 1) {
    currentRevealIndex.value++
    revealStep.value = 1
  }
}

function startReveal() {
  currentRevealIndex.value = 0
  revealStep.value = 1
  isSetupVisible.value = false
}
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

h1 {
  text-align: center;
  color: #2c3e50;
}

.form-section {
  margin-bottom: 30px;
}

.families-section {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 20px;
  margin-bottom: 30px;
}

.family-card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 15px;
  background: #f9f9f9;
}

.member-form {
  display: flex;
  gap: 10px;
  margin-bottom: 10px;
}

input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  flex: 1;
}

button {
  padding: 8px 15px;
  border: none;
  border-radius: 4px;
  background: #4CAF50;
  color: white;
  cursor: pointer;
}

button:hover {
  background: #45a049;
}

button:disabled {
  background: #cccccc;
  cursor: not-allowed;
}

.delete-btn {
  background: #ff4444;
  padding: 2px 6px;
  margin-left: 5px;
}

.delete-family {
  background: #ff4444;
  width: 100%;
  margin-top: 10px;
}

.actions {
  display: flex;
  gap: 10px;
  justify-content: center;
  margin: 20px 0;
  flex-wrap: wrap;
}

.primary-btn {
  background: #2196F3;
  padding: 10px 20px;
}

.secondary-btn {
  background: #9e9e9e;
  padding: 10px 20px;
}

.warning-btn {
  background: #ff4444;
  padding: 10px 20px;
}

.warning-btn:hover {
  background: #ff0000;
}

.reveal-section {
  text-align: center;
  margin-top: 30px;
}

.assignment-card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 20px;
  margin: 20px auto;
  max-width: 300px;
  background: white;
}

.reveal-btn {
  background: #673AB7;
  padding: 10px 20px;
  margin-bottom: 15px;
}

.assignment {
  font-size: 1.2em;
}

ul {
  list-style: none;
  padding: 0;
}

li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 5px 0;
}

.reset-buttons {
  display: flex;
  gap: 10px;
  margin-left: 10px;
}

.reveal-step {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
}

.name-display {
  font-size: 1.5em;
  font-weight: bold;
  color: #2196F3;
  margin: 10px 0;
}

.reveal-step h3 {
  margin: 0;
  color: #2c3e50;
}

.reveal-step p {
  margin: 5px 0;
}

.toggle-btn {
  display: block;
  margin: 0 auto 20px;
  background: #673AB7;
  padding: 8px 16px;
  border-radius: 4px;
  color: white;
  cursor: pointer;
  transition: background-color 0.3s;
}

.toggle-btn:hover {
  background: #5c34a4;
}
</style>
