<script setup>
import { ref, reactive, computed } from 'vue';

const showModal = ref(false);
const showInitiativeModal = ref(false);
const topCards = ref([]);
const initiativeValues = ref({});
const selectedCardIndex = ref(null);
const activeCardIndex = ref(null);
const isEditing = ref(false);
const scrollContainer = ref(null);

const handleWheel = (e) => {
  if (scrollContainer.value) {
    scrollContainer.value.scrollLeft += e.deltaY;
  }
};

const getModifier = (score) => {
  if (score === '' || score === null || score === undefined) return '';
  const val = Number(score);
  if (isNaN(val)) return '';
  const mod = Math.floor((val - 10) / 2);
  return mod >= 0 ? `+${mod}` : `${mod}`;
};

// Hit Calculator State
const hitCalc = reactive({
  d20: '',
  mod: 0,
  ac: ''
});

const hitResult = computed(() => {
  const d20 = Number(hitCalc.d20);
  const mod = Number(hitCalc.mod);
  const ac = Number(hitCalc.ac);

  if (!hitCalc.d20 || hitCalc.ac === '') return null;

  if (d20 === 20) return { text: 'Critical Hit (暴击)', class: 'crit-hit', isHit: true, isCrit: true };
  if (d20 === 1) return { text: 'Critical Miss (大失败)', class: 'crit-miss', isHit: false, isCrit: false };
  
  const total = d20 + mod;
  if (total >= ac) return { text: `Hit (命中) - Total: ${total}`, class: 'hit', isHit: true, isCrit: false };
  return { text: `Miss (未命中) - Total: ${total}`, class: 'miss', isHit: false, isCrit: false };
});

// Damage Calculator State
const showDamageCalc = ref(false);
const damageCalc = reactive({
  baseDamage: '',
  bonusDamage: '',
  isCrit: false
});

const damageResult = computed(() => {
  const base = Number(damageCalc.baseDamage) || 0;
  const bonus = Number(damageCalc.bonusDamage) || 0;
  
  // DND 5e Crit Rule: Roll damage dice twice, then add modifier.
  // Since we take manual input, if it's a crit, we assume user might enter dice total.
  // But typically for a helper, we just sum them.
  // If we wanted to automate crit doubling, we'd need dice inputs (e.g. 2d6).
  // For simple manual entry: Total = Base + Bonus. 
  // If IsCrit is true, visually emphasize.
  
  return base + bonus;
});

const openDamageCalc = () => {
  if (hitResult.value && hitResult.value.isHit) {
    showDamageCalc.value = true;
    damageCalc.baseDamage = '';
    damageCalc.bonusDamage = '';
    damageCalc.isCrit = hitResult.value.isCrit;
  }
};

const applyDamage = () => {
  if (selectedCardIndex.value !== null && topCards.value[selectedCardIndex.value]) {
    const damage = damageResult.value;
    const currentHp = topCards.value[selectedCardIndex.value].hp;
    topCards.value[selectedCardIndex.value].hp = Math.max(0, currentHp - damage);
  } else {
    alert('Please select a target unit (click a card) to apply damage.');
    return;
  }
  showDamageCalc.value = false;
};

const closeDamageCalc = () => {
  showDamageCalc.value = false;
};

const initialFormState = {
  name: '',
  avatar: '', // URL for avatar image
  faction: 'Neutral', // Default faction
  str: '',
  dex: '',
  con: '',
  int: '',
  wis: '',
  cha: '',
  ac: '',
  hp: '',
  maxHp: '',
  attackBonus: '',
  damageBonus: '',
  saveBonus: '',
  skillBonus: '',
  proficiencyBonus: '',
  speed: '',
  skills: []
};

const skillOptions = [
  '运动 (Athletics)', '体操 (Acrobatics)', '巧手 (Sleight of Hand)',
  '隐匿 (Stealth)', '察觉 (Perception)', '生存 (Survival)',
  '医疗 (Medicine)', '洞悉 (Insight)', '欺瞒 (Deception)',
  '威吓 (Intimidation)', '表演 (Performance)', '游说 (Persuasion)',
  '奥秘 (Arcana)', '历史 (History)', '调查 (Investigation)',
  '自然 (Nature)', '宗教 (Religion)', '驯兽 (Animal Handling)'
];

const toggleSkill = (card, skill) => {
  if (!card.skills) card.skills = [];
  const index = card.skills.indexOf(skill);
  if (index === -1) {
    card.skills.push(skill);
  } else {
    card.skills.splice(index, 1);
  }
};

const formData = reactive({ ...initialFormState });

const openModal = () => {
  isEditing.value = false;
  Object.assign(formData, initialFormState);
  showModal.value = true;
};

const editCard = () => {
  if (selectedCardIndex.value === null) {
    alert('Please select a card to edit.');
    return;
  }
  isEditing.value = true;
  Object.assign(formData, topCards.value[selectedCardIndex.value]);
  showModal.value = true;
};

const selectCard = (index) => {
  selectedCardIndex.value = selectedCardIndex.value === index ? null : index;
};

const selectNextCard = () => {
  if (topCards.value.length === 0) return;
  
  if (activeCardIndex.value === null) {
    activeCardIndex.value = 0;
  } else {
    activeCardIndex.value = (activeCardIndex.value + 1) % topCards.value.length;
  }
};

const deleteCard = () => {
  if (selectedCardIndex.value === null) {
    alert('Please select a card to delete.');
    return;
  }
  
  const deletedIndex = selectedCardIndex.value;
  topCards.value.splice(deletedIndex, 1);
  
  // Update Active Card Index
  if (activeCardIndex.value !== null) {
    if (activeCardIndex.value === deletedIndex) {
      // If the active card was deleted
      if (topCards.value.length === 0) {
        activeCardIndex.value = null;
      } else {
        // Move to next (or wrap to 0 if we deleted the last one)
        // If deletedIndex was the last element, activeCardIndex (which equaled deletedIndex)
        // is now out of bounds (== length). So we set it to 0.
        // If deletedIndex was not last, the next element shifted into this index, so we stay.
        if (activeCardIndex.value >= topCards.value.length) {
          activeCardIndex.value = 0;
        }
      }
    } else if (activeCardIndex.value > deletedIndex) {
      // If active card was after the deleted one, shift it down
      activeCardIndex.value--;
    }
  }

  // Update Selected Card Index (Keep existing logic of cycling/selecting next)
  if (topCards.value.length === 0) {
    selectedCardIndex.value = null;
  } else {
    if (deletedIndex >= topCards.value.length) {
      selectedCardIndex.value = 0;
    } else {
      selectedCardIndex.value = deletedIndex;
    }
  }
};

const confirmAdd = () => {
  if (formData.name.trim() === '') {
    alert('Please enter a name.');
    return;
  }
  
  // Simple validation to ensure all numeric fields are integers
  for (const key in formData) {
    if (['name', 'initiative', 'avatar', 'faction'].includes(key)) continue;
    if (formData[key] === '') continue; // Allow empty fields during check, will handle specific logic later if needed
    if (!Number.isInteger(Number(formData[key]))) {
      alert('Please enter valid integers for all numeric fields.');
      return;
    }
  }

  // Handle MaxHP logic
  if (formData.hp !== '' && (formData.maxHp === '' || formData.maxHp === null || formData.maxHp === undefined)) {
    formData.maxHp = formData.hp;
  }

  if (isEditing.value && selectedCardIndex.value !== null) {
    Object.assign(topCards.value[selectedCardIndex.value], formData);
  } else {
    topCards.value.push({ ...formData });
  }
  showModal.value = false;
};

const openInitiativeModal = () => {
  if (topCards.value.length === 0) {
    alert('No cards available to set initiative.');
    return;
  }
  initiativeValues.value = {};
  topCards.value.forEach((card, index) => {
    initiativeValues.value[index] = '';
  });
  showInitiativeModal.value = true;
};

const confirmInitiative = () => {
  // Validate initiative values
  for (const index in initiativeValues.value) {
    if (initiativeValues.value[index] === '' || !Number.isInteger(Number(initiativeValues.value[index]))) {
      alert('Please enter valid integer initiative values for all cards.');
      return;
    }
  }

  // Update cards with initiative and sort
  const sortedCards = topCards.value.map((card, index) => ({
    ...card,
    initiative: Number(initiativeValues.value[index])
  })).sort((a, b) => b.initiative - a.initiative);

  topCards.value = sortedCards;
  activeCardIndex.value = 0; // Reset active card to the first one in new order
  selectedCardIndex.value = null; // Clear selection
  showInitiativeModal.value = false;
};

const quickAdd = () => {
  const defaultCard = {
    name: 'Default',
    avatar: '',
    faction: 'Neutral',
    str: 10,
    dex: 10,
    con: 10,
    int: 10,
    wis: 10,
    cha: 10,
    ac: 10,
    hp: 10,
    maxHp: 10,
    attackBonus: 0,
    damageBonus: 0,
    saveBonus: 0,
    skillBonus: 0,
    proficiencyBonus: 2,
    speed: 30,
    skills: []
  };
  topCards.value.push(defaultCard);
};
</script>

<template>
  <div class="container">
    <div class="section top-container">
      <div 
        class="card-list" 
        ref="scrollContainer"
        @wheel.prevent="handleWheel"
      >
        <div 
          v-for="(card, index) in topCards" 
          :key="index" 
          class="card"
          :class="{ 
            selected: index === selectedCardIndex, 
            active: index === activeCardIndex,
            downed: card.hp <= 0
          }"
          @click="selectCard(index)"
        >
          <div v-if="card.hp <= 0" class="downed-status">倒地</div>
          <div class="card-header">
             <div class="avatar-container">
                <img v-if="card.avatar" :src="card.avatar" alt="Avatar" class="card-avatar" />
                <div v-else class="avatar-placeholder">{{ card.name ? card.name.charAt(0).toUpperCase() : '?' }}</div>
             </div>
             <div class="card-name">{{ card.name }}</div>
          </div>
          <div class="card-stats">
            <div v-if="card.initiative !== undefined" class="card-initiative">Init: {{ card.initiative }}</div>
            <div class="card-hp-container">
              <div class="hp-text">HP: {{ card.hp }} / {{ card.maxHp || card.hp }}</div>
              <div class="hp-bar-bg">
                <div 
                  class="hp-bar-fill" 
                  :style="{ width: Math.min(100, Math.max(0, (card.hp / (card.maxHp || card.hp || 1)) * 100)) + '%' }"
                ></div>
              </div>
            </div>
          </div>
        </div>
      </div>
      
      <!-- Operations Buttons (Moved to Top Right) -->
      <div class="top-actions">
        <div class="button-grid">
          <button class="btn add small-btn" @click="openModal" title="Add Unit">+</button>
          <button class="btn quick-add small-btn" @click="quickAdd" title="Quick Add">⚡</button>
          <button class="btn delete small-btn" @click="deleteCard" title="Delete Selected">🗑️</button>
          <button class="btn initiative small-btn" @click="openInitiativeModal" title="Set Initiative">Init</button>
          <button class="btn next small-btn" @click="selectNextCard" title="Next Turn">Next</button>
        </div>
      </div>
    </div>
    <div class="section middle">
      <!-- Left Panel: Active Card Info -->
      <div class="panel left-panel">
        <div v-if="activeCardIndex !== null && topCards[activeCardIndex]" class="detail-view active-view">
          <h3>Current Turn</h3>
          <div class="detail-content">
            <div class="detail-header editable-header">
              <input 
                v-model="topCards[activeCardIndex].name" 
                class="edit-input name-input" 
                placeholder="Name" 
              />
              <select v-model="topCards[activeCardIndex].faction" class="edit-select faction-select">
                <option value="Friendly">Friendly</option>
                <option value="Neutral">Neutral</option>
                <option value="Hostile">Hostile</option>
              </select>
              <div class="hp-edit-container">
                <span class="label">HP:</span>
                <input 
                  type="number" 
                  v-model.number="topCards[activeCardIndex].hp" 
                  class="edit-input hp-input" 
                />
                <span class="separator">/</span>
                <input 
                  type="number" 
                  v-model.number="topCards[activeCardIndex].maxHp" 
                  class="edit-input hp-input" 
                  placeholder="Max" 
                />
              </div>
            </div>
            <div class="detail-stats editable-stats">
              <div class="stat-item">
                <span class="label">AC:</span>
                <input 
                  type="number" 
                  v-model.number="topCards[activeCardIndex].ac" 
                  class="edit-input stat-input" 
                />
              </div>
              <div class="stat-item">
                <span class="label">Init:</span>
                <input 
                  type="number" 
                  v-model.number="topCards[activeCardIndex].initiative" 
                  class="edit-input stat-input" 
                />
              </div>
            </div>
            <div class="detail-attributes editable-attributes">
              <div class="attr-row">
                <span class="attr-label">STR:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].str" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[activeCardIndex].str) }}</span>
              </div>
              <div class="attr-row">
                <span class="attr-label">DEX:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].dex" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[activeCardIndex].dex) }}</span>
              </div>
              <div class="attr-row">
                <span class="attr-label">CON:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].con" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[activeCardIndex].con) }}</span>
              </div>
              <div class="attr-row">
                <span class="attr-label">INT:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].int" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[activeCardIndex].int) }}</span>
              </div>
              <div class="attr-row">
                <span class="attr-label">WIS:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].wis" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[activeCardIndex].wis) }}</span>
              </div>
              <div class="attr-row">
                <span class="attr-label">CHA:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].cha" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[activeCardIndex].cha) }}</span>
              </div>
            </div>
            <div class="detail-extras editable-extras">
              <div class="extra-row">
                <span class="extra-label">Speed:</span>
                <div class="input-unit-group">
                  <input type="number" v-model.number="topCards[activeCardIndex].speed" class="edit-input extra-input" />
                  <span class="unit">ft</span>
                </div>
              </div>
              <div class="extra-row">
                <span class="extra-label">Prof:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].proficiencyBonus" class="edit-input extra-input" />
              </div>
              <div class="extra-row">
                <span class="extra-label">Atk Bonus:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].attackBonus" class="edit-input extra-input" />
              </div>
              <div class="extra-row">
                <span class="extra-label">Dmg Bonus:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].damageBonus" class="edit-input extra-input" />
              </div>
              <div class="extra-row">
                <span class="extra-label">Save Bonus:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].saveBonus" class="edit-input extra-input" />
              </div>
              <div class="extra-row">
                <span class="extra-label">Skill Bonus:</span>
                <input type="number" v-model.number="topCards[activeCardIndex].skillBonus" class="edit-input extra-input" />
              </div>
            </div>

            <!-- Skills Section -->
            <div class="skills-section">
              <h4>Skills</h4>
              <div class="skills-container">
                <div 
                  v-for="skill in skillOptions" 
                  :key="skill"
                  class="skill-chip"
                  :class="{ selected: topCards[activeCardIndex].skills && topCards[activeCardIndex].skills.includes(skill) }"
                  @click="toggleSkill(topCards[activeCardIndex], skill)"
                >
                  {{ skill.split(' ')[0] }}
                </div>
              </div>
            </div>
          </div>
        </div>
        <div v-else class="empty-state">No Active Unit</div>
      </div>

      <!-- Center Panel: Hit Check & Damage Calculator -->
      <div class="panel center-panel">
        <template v-if="!showDamageCalc">
          <h3>攻击检定</h3>
          <div class="hit-calculator">
            <div class="form-group-small">
              <label>d20 Roll (1-20):</label>
              <input type="number" v-model.number="hitCalc.d20" placeholder="Roll" min="1" max="20" />
            </div>
            <div class="form-group-small">
              <label>修正值 (+/-):</label>
              <input type="number" v-model.number="hitCalc.mod" placeholder="Mod" />
            </div>
            <div class="form-group-small">
              <label>目标 AC:</label>
              <input type="number" v-model.number="hitCalc.ac" placeholder="AC" />
            </div>
            
            <div 
              v-if="hitResult" 
              class="hit-result clickable-result" 
              :class="hitResult.class"
              @click="openDamageCalc"
              title="Click to Calculate Damage"
            >
              {{ hitResult.text }} <span v-if="hitResult.isHit" class="click-hint">➜ Damage</span>
            </div>
            <div v-else class="hit-placeholder">
              Enter values to check hit
            </div>
          </div>
        </template>
        
        <template v-else>
          <h3>伤害计算 <span class="back-btn" @click="closeDamageCalc">↩</span></h3>
          <div class="damage-calculator">
            <div v-if="damageCalc.isCrit" class="crit-badge">Critical Hit! (Dice Doubled)</div>
            <div class="form-group-small">
              <label>基础伤害:</label>
              <input type="number" v-model.number="damageCalc.baseDamage" placeholder="Dice Roll" />
            </div>
            <div class="form-group-small">
              <label>修正伤害:</label>
              <input type="number" v-model.number="damageCalc.bonusDamage" placeholder="Mod/Magic" />
            </div>
            
            <div class="damage-total">
              Total: <span class="total-val">{{ damageResult }}</span>
            </div>
            
            <button class="btn apply-dmg" @click="applyDamage">Done</button>
          </div>
        </template>
      </div>

      <!-- Right Panel: Selected Card Info -->
      <div class="panel right-panel">
        <div v-if="selectedCardIndex !== null && topCards[selectedCardIndex]" class="detail-view selected-view">
          <h3>Selected Unit</h3>
          <div class="detail-content">
            <div class="detail-header editable-header">
              <input 
                v-model="topCards[selectedCardIndex].name" 
                class="edit-input name-input" 
                placeholder="Name" 
              />
              <select v-model="topCards[selectedCardIndex].faction" class="edit-select faction-select">
                <option value="Friendly">Friendly</option>
                <option value="Neutral">Neutral</option>
                <option value="Hostile">Hostile</option>
              </select>
              <div class="hp-edit-container">
                <span class="label">HP:</span>
                <input 
                  type="number" 
                  v-model.number="topCards[selectedCardIndex].hp" 
                  class="edit-input hp-input" 
                />
                <span class="separator">/</span>
                <input 
                  type="number" 
                  v-model.number="topCards[selectedCardIndex].maxHp" 
                  class="edit-input hp-input" 
                  placeholder="Max" 
                />
              </div>
            </div>
            <div class="detail-stats editable-stats">
              <div class="stat-item">
                <span class="label">AC:</span>
                <input 
                  type="number" 
                  v-model.number="topCards[selectedCardIndex].ac" 
                  class="edit-input stat-input" 
                />
              </div>
              <div class="stat-item">
                <span class="label">Init:</span>
                <input 
                  type="number" 
                  v-model.number="topCards[selectedCardIndex].initiative" 
                  class="edit-input stat-input" 
                />
              </div>
            </div>
            <div class="detail-attributes editable-attributes">
              <div class="attr-row">
                <span class="attr-label">STR:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].str" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[selectedCardIndex].str) }}</span>
              </div>
              <div class="attr-row">
                <span class="attr-label">DEX:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].dex" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[selectedCardIndex].dex) }}</span>
              </div>
              <div class="attr-row">
                <span class="attr-label">CON:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].con" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[selectedCardIndex].con) }}</span>
              </div>
              <div class="attr-row">
                <span class="attr-label">INT:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].int" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[selectedCardIndex].int) }}</span>
              </div>
              <div class="attr-row">
                <span class="attr-label">WIS:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].wis" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[selectedCardIndex].wis) }}</span>
              </div>
              <div class="attr-row">
                <span class="attr-label">CHA:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].cha" class="edit-input attr-input" />
                <span class="mod-text">{{ getModifier(topCards[selectedCardIndex].cha) }}</span>
              </div>
            </div>
            <div class="detail-extras editable-extras">
              <div class="extra-row">
                <span class="extra-label">Speed:</span>
                <div class="input-unit-group">
                  <input type="number" v-model.number="topCards[selectedCardIndex].speed" class="edit-input extra-input" />
                  <span class="unit">ft</span>
                </div>
              </div>
              <div class="extra-row">
                <span class="extra-label">Prof:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].proficiencyBonus" class="edit-input extra-input" />
              </div>
              <div class="extra-row">
                <span class="extra-label">Atk Bonus:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].attackBonus" class="edit-input extra-input" />
              </div>
              <div class="extra-row">
                <span class="extra-label">Dmg Bonus:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].damageBonus" class="edit-input extra-input" />
              </div>
              <div class="extra-row">
                <span class="extra-label">Save Bonus:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].saveBonus" class="edit-input extra-input" />
              </div>
              <div class="extra-row">
                <span class="extra-label">Skill Bonus:</span>
                <input type="number" v-model.number="topCards[selectedCardIndex].skillBonus" class="edit-input extra-input" />
              </div>
            </div>

            <!-- Skills Section -->
            <div class="skills-section">
              <h4>Skills</h4>
              <div class="skills-container">
                <div 
                  v-for="skill in skillOptions" 
                  :key="skill"
                  class="skill-chip"
                  :class="{ selected: topCards[selectedCardIndex].skills && topCards[selectedCardIndex].skills.includes(skill) }"
                  @click="toggleSkill(topCards[selectedCardIndex], skill)"
                >
                  {{ skill.split(' ')[0] }}
                </div>
              </div>
            </div>
          </div>
        </div>
        <div v-else class="empty-state">Select a unit to view details</div>
      </div>
    </div>
  </div>

  <div v-if="showModal" class="modal-overlay">
    <div class="modal-content">
      <h3>{{ isEditing ? 'Edit Character Stats' : 'Add Character Stats' }}</h3>
      <div class="form-group">
        <label>Name (姓名):</label>
        <input type="text" v-model="formData.name" />
      </div>
      <div class="form-group">
        <label>Avatar URL (头像链接):</label>
        <input type="text" v-model="formData.avatar" placeholder="https://example.com/image.png" />
      </div>
      <div class="form-group">
        <label>Faction (阵营):</label>
        <select v-model="formData.faction" class="form-select">
          <option value="Friendly">Friendly (友善)</option>
          <option value="Neutral">Neutral (中立)</option>
          <option value="Hostile">Hostile (敌对)</option>
        </select>
      </div>
      <div class="form-group">
        <label>Strength (力量): <span class="mod-preview" v-if="formData.str !== ''">{{ getModifier(formData.str) }}</span></label>
        <input type="number" v-model.number="formData.str" />
      </div>
      <div class="form-group">
        <label>Dexterity (敏捷): <span class="mod-preview" v-if="formData.dex !== ''">{{ getModifier(formData.dex) }}</span></label>
        <input type="number" v-model.number="formData.dex" />
      </div>
      <div class="form-group">
        <label>Constitution (体质): <span class="mod-preview" v-if="formData.con !== ''">{{ getModifier(formData.con) }}</span></label>
        <input type="number" v-model.number="formData.con" />
      </div>
      <div class="form-group">
        <label>Intelligence (智力): <span class="mod-preview" v-if="formData.int !== ''">{{ getModifier(formData.int) }}</span></label>
        <input type="number" v-model.number="formData.int" />
      </div>
      <div class="form-group">
        <label>Wisdom (感知): <span class="mod-preview" v-if="formData.wis !== ''">{{ getModifier(formData.wis) }}</span></label>
        <input type="number" v-model.number="formData.wis" />
      </div>
      <div class="form-group">
        <label>Charisma (魅力): <span class="mod-preview" v-if="formData.cha !== ''">{{ getModifier(formData.cha) }}</span></label>
        <input type="number" v-model.number="formData.cha" />
      </div>
      <div class="form-group">
        <label>AC (Armor Class):</label>
        <input type="number" v-model.number="formData.ac" />
      </div>
      <div class="form-group">
        <label>Current HP (当前生命值):</label>
        <input type="number" v-model.number="formData.hp" />
      </div>
      <div class="form-group">
        <label>Max HP (最大生命值):</label>
        <input type="number" v-model.number="formData.maxHp" placeholder="Leave blank to match HP" />
      </div>
      <div class="form-group">
        <label>Speed (速度 ft):</label>
        <input type="number" v-model.number="formData.speed" />
      </div>
      <div class="form-group">
        <label>Skills (技能):</label>
        <div class="skills-container">
          <div 
            v-for="skill in skillOptions" 
            :key="skill"
            class="skill-chip"
            :class="{ selected: formData.skills && formData.skills.includes(skill) }"
            @click="toggleSkill(formData, skill)"
          >
            {{ skill.split(' ')[0] }}
          </div>
        </div>
      </div>
      <div class="modal-actions">
        <button class="btn confirm" @click="confirmAdd">Confirm</button>
        <button class="btn cancel" @click="showModal = false">Cancel</button>
      </div>
    </div>
  </div>

  <div v-if="showInitiativeModal" class="modal-overlay">
    <div class="modal-content">
      <h3>Set Initiative Values</h3>
      <div v-for="(card, index) in topCards" :key="index" class="form-group">
        <label>{{ card.name }} (Initiative):</label>
        <input type="number" v-model.number="initiativeValues[index]" />
      </div>
      <div class="modal-actions">
        <button class="btn confirm" @click="confirmInitiative">Sort</button>
        <button class="btn cancel" @click="showInitiativeModal = false">Cancel</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  height: 100%;
  width: 100%;
  margin: 0;
  padding: 0;
}

.section {
  width: 100%;
}

.top-container {
  flex: 1; /* 25% */
  background-color: #4CAF50; /* Green */
  display: flex;
  flex-direction: row;
  overflow: hidden; /* Manage scroll internally */
}

.card-list {
  flex: 1;
  display: flex;
  flex-wrap: nowrap;
  align-items: center;
  padding: 10px;
  gap: 10px;
  overflow-x: auto;
  overflow-y: hidden;
  white-space: nowrap;
}

.top-actions {
  flex: 0 0 auto;
  padding: 10px;
  display: flex;
  align-items: center;
  background-color: rgba(0, 0, 0, 0.1);
  border-left: 1px solid rgba(255, 255, 255, 0.2);
}

.button-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 5px;
}

.small-btn {
  padding: 5px 10px;
  font-size: 0.9em;
  min-width: 40px;
}

.card {
  background-color: white;
  border-radius: 8px;
  padding: 10px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  min-width: 150px;
  width: 150px;
  font-size: 0.9em;
  flex-shrink: 0;
  cursor: pointer;
  border: 2px solid transparent;
  transition: border-color 0.2s;
  position: relative;
}

.card.selected {
  border-color: #2196F3;
}

.card.active {
  border-color: #9C27B0;
  box-shadow: 0 0 8px rgba(156, 39, 176, 0.6);
}

.card.selected.active {
  border-color: #9C27B0; /* Active state visual priority */
  box-shadow: 0 0 0 2px #2196F3, 0 0 12px rgba(156, 39, 176, 0.6);
}

.card-name {
  font-weight: bold;
  text-align: center;
  margin-bottom: 5px;
  border-bottom: 1px solid #eee;
  padding-bottom: 5px;
}

.card-initiative {
  text-align: center;
  color: #f44336;
  font-weight: bold;
  margin-bottom: 5px;
}

.card-hp-container {
  width: 100%;
}

.hp-text {
  text-align: center;
  font-weight: bold;
  margin-bottom: 2px;
  font-size: 0.9em;
}

.hp-bar-bg {
  width: 100%;
  height: 8px;
  background-color: #ffebee;
  border-radius: 4px;
  border: 1px solid #ffcdd2;
  overflow: hidden;
}

.hp-bar-fill {
  height: 100%;
  background-color: #4CAF50;
  transition: width 0.3s ease;
}

/* Editable Inputs Styling */
.edit-input, .edit-select {
  border: 1px solid transparent;
  background-color: transparent;
  font-family: inherit;
  font-size: inherit;
  padding: 2px 4px;
  border-radius: 4px;
  transition: all 0.2s;
}

.edit-input:hover, .edit-select:hover {
  background-color: rgba(0, 0, 0, 0.05);
  border-color: #ddd;
}

.edit-input:focus, .edit-select:focus {
  outline: none;
  background-color: white;
  border-color: #2196F3;
  box-shadow: 0 0 0 2px rgba(33, 150, 243, 0.2);
}

.editable-header {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 15px;
}

.name-input {
  font-weight: bold;
  font-size: 1.2em;
  text-align: center;
  width: 100%;
}

.faction-select {
  text-align: center;
  margin: 0 auto;
  display: block;
  cursor: pointer;
}

.hp-edit-container {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 5px;
}

.hp-input {
  width: 50px;
  text-align: center;
  font-weight: bold;
}

.editable-stats {
  display: flex;
  justify-content: space-around;
  margin-bottom: 15px;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 5px;
}

.stat-input {
  width: 40px;
  text-align: center;
}

/* Updated Middle Section Layout */
.middle {
  flex: 2;
  background-color: white;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  padding: 10px;
  gap: 10px;
  overflow-x: auto; /* Allow horizontal scroll if panels don't fit */
  overflow-y: hidden; /* Panels handle vertical scroll */
}

.panel {
  flex: 1;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  padding: 15px;
  background-color: #f9f9f9;
  display: flex;
  flex-direction: column;
  min-width: 300px; /* Ensure minimum width for readability */
  overflow-y: auto; /* Internal scrolling */
  box-shadow: 0 2px 5px rgba(0,0,0,0.05);
}

/* Compact Attribute Grid (Stat Boxes) */
.editable-attributes {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
  margin-bottom: 15px;
}

.attr-row {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background-color: #ffffff;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  padding: 5px;
  box-shadow: 0 1px 2px rgba(0,0,0,0.05);
}

.attr-label {
  font-size: 0.75em;
  font-weight: bold;
  color: #666;
  text-transform: uppercase;
  margin-bottom: 2px;
  width: auto; /* Reset fixed width */
}

.attr-input {
  width: 100%;
  text-align: center;
  font-size: 1.2em;
  font-weight: bold;
  color: #333;
  border: none;
  border-bottom: 2px solid #eee;
  background: transparent;
  padding: 2px 0;
}

.attr-input:focus {
  border-color: #2196F3;
  outline: none;
}

.mod-text {
  font-size: 0.85em;
  color: #888;
  margin-top: 2px;
  font-weight: bold;
}

/* Compact Extras Grid */
.detail-extras {
  margin-top: 10px;
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
  border-top: 1px solid #eee;
  padding-top: 10px;
}

.extra-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 0.85em;
  background-color: white;
  padding: 4px 8px;
  border-radius: 4px;
  border: 1px solid #eee;
}

.extra-label {
  font-weight: bold;
  color: #555;
}

.extra-input {
  width: 50px;
  text-align: right;
  border: none;
  border-bottom: 1px solid #ddd;
  background: transparent;
  font-weight: bold;
}

/* Header Adjustments */
.detail-header {
  margin-bottom: 10px;
  padding-bottom: 10px;
}

.name-input {
  font-size: 1.3em;
  margin-bottom: 5px;
  border-bottom: 1px solid #eee;
}

.hp-edit-container {
  margin-top: 5px;
  font-size: 1.1em;
}

.hp-input {
  width: 60px;
  font-size: 1em;
}

.editable-stats {
  justify-content: center;
  gap: 20px;
}

.stat-item {
  flex-direction: column;
  background: white;
  padding: 5px 10px;
  border-radius: 6px;
  border: 1px solid #eee;
  min-width: 60px;
}

.stat-item .label {
  font-size: 0.75em;
  text-transform: uppercase;
  color: #666;
  font-weight: bold;
}

.stat-input {
  font-size: 1.2em;
  font-weight: bold;
  width: 100%;
  border: none;
  background: transparent;
}

/* Visual Polish */
.input-unit-group {
  display: flex;
  align-items: center;
  gap: 2px;
}

.unit {
  color: #888;
  font-size: 0.8em;
}

.left-panel {
  border-top: 4px solid #4CAF50; /* Green accent for Current Turn */
}

.right-panel {
  border-top: 4px solid #2196F3; /* Blue accent for Selected Unit */
}

.detail-view h3 {
  margin-top: 0;
  margin-bottom: 15px;
  text-align: center;
  font-size: 1.1em;
  color: #444;
  text-transform: uppercase;
  letter-spacing: 1px;
  border-bottom: 2px solid #eee;
  padding-bottom: 10px;
}

.center-panel {
  flex: 0 0 250px; /* Slightly wider for calculator */
  background-color: #fff;
  border: none;
}

.hit-calculator {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.form-group-small {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.form-group-small label {
  font-size: 0.9em;
  font-weight: bold;
}

.form-group-small input {
  width: 80px;
  padding: 5px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.hit-result {
  margin-top: 10px;
  padding: 10px;
  text-align: center;
  font-weight: bold;
  border-radius: 4px;
  color: white;
  transition: transform 0.1s;
}

.clickable-result {
  cursor: pointer;
  position: relative;
}

.clickable-result:active {
  transform: scale(0.98);
}

.click-hint {
  font-size: 0.8em;
  opacity: 0.8;
  margin-left: 5px;
}

.damage-calculator {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.crit-badge {
  background-color: #9C27B0;
  color: white;
  font-weight: bold;
  text-align: center;
  padding: 5px;
  border-radius: 4px;
  font-size: 0.9em;
}

.damage-total {
  font-size: 1.2em;
  font-weight: bold;
  text-align: center;
  margin: 10px 0;
  padding: 10px;
  background-color: #f1f1f1;
  border-radius: 4px;
}

.total-val {
  color: #f44336;
  font-size: 1.4em;
}

.apply-dmg {
  background-color: #f44336;
  color: white;
  width: 100%;
}

.back-btn {
  cursor: pointer;
  float: right;
  font-size: 1.2em;
}

.card.downed {
  opacity: 0.8;
  border: 2px solid #f44336;
  background-color: #ffebee;
}

.downed-status {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%) rotate(-20deg);
  font-size: 1.5em;
  font-weight: 900;
  color: #c62828;
  border: 3px solid #c62828;
  padding: 2px 8px;
  border-radius: 4px;
  z-index: 5;
  background-color: rgba(255, 255, 255, 0.9);
  pointer-events: none;
  box-shadow: 0 2px 5px rgba(0,0,0,0.2);
}

.hit-placeholder {
  margin-top: 10px;
  text-align: center;
  color: #999;
  font-style: italic;
  font-size: 0.9em;
}

.hit {
  background-color: #4CAF50;
}

.miss {
  background-color: #f44336;
}

.crit-hit {
  background-color: #9C27B0;
  box-shadow: 0 0 10px #9C27B0;
}

.crit-miss {
  background-color: #607D8B;
}

.button-group-vertical {
  display: flex;
  flex-direction: column;
  gap: 10px;
  width: 100%;
}

.button-group-vertical .btn {
  width: 100%;
}

.detail-view {
  display: flex;
  flex-direction: column;
  gap: 10px;
  height: 100%;
}

.detail-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 2px solid #ccc;
  padding-bottom: 10px;
  margin-bottom: 10px;
}

.detail-name {
  font-size: 1.5em;
  font-weight: bold;
}

.detail-hp {
  font-size: 1.2em;
  font-weight: bold;
  color: #4CAF50;
}

.detail-stats, .detail-attributes {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.detail-attributes {
  margin-top: 10px;
  border-top: 1px solid #eee;
  padding-top: 10px;
}

.empty-state {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  color: #999;
  font-style: italic;
}

.btn {
  padding: 10px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: opacity 0.2s;
}

.btn:hover {
  opacity: 0.9;
}

.add {
  background-color: #4CAF50;
  color: white;
}

.next {
  background-color: #9C27B0; /* Purple */
  color: white;
}

.delete {
  background-color: #f44336;
  color: white;
}

.edit {
  background-color: #FFC107; /* Amber */
  color: black;
}

.initiative {
  background-color: #2196F3; /* Blue */
  color: white;
}

.quick-add {
  background-color: #FF9800; /* Orange */
  color: white;
}

/* .bottom removed */

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background-color: white;
  padding: 20px;
  border-radius: 8px;
  width: 300px;
  max-height: 90vh;
  overflow-y: auto;
}

.form-group {
  margin-bottom: 10px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

.form-group input {
  width: 100%;
  padding: 5px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.modal-actions {
  display: flex;
  justify-content: space-between;
  margin-top: 20px;
}

.confirm {
  background-color: #4CAF50;
  color: white;
}

.cancel {
  background-color: #999;
  color: white;
}

/* New Styles */
.card-header {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
  margin-bottom: 5px;
  border-bottom: 1px solid #eee;
  padding-bottom: 5px;
}

.avatar-container {
  width: 50px;
  height: 50px;
  border-radius: 50%;
  overflow: hidden;
  background-color: #eee;
  display: flex;
  justify-content: center;
  align-items: center;
  border: 2px solid #ddd;
}

.card-avatar {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.avatar-placeholder {
  font-size: 24px;
  font-weight: bold;
  color: #888;
}

.card-stats {
  display: flex;
  justify-content: space-around;
  font-size: 0.9em;
}

.card-hp {
  color: #4CAF50;
  font-weight: bold;
}

.detail-faction {
  font-weight: bold;
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 0.8em;
  text-transform: uppercase;
}

.detail-faction.friendly {
  background-color: #e8f5e9;
  color: #2e7d32;
  border: 1px solid #a5d6a7;
}

.detail-faction.neutral {
  background-color: #fff3e0;
  color: #ef6c00;
  border: 1px solid #ffcc80;
}

.detail-faction.hostile {
  background-color: #ffebee;
  color: #c62828;
  border: 1px solid #ef9a9a;
}

.form-select {
  width: 100%;
  padding: 5px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.mod-text {
  color: #777;
  font-weight: normal;
  margin-left: 5px;
}

.mod-preview {
  color: #2196F3;
  margin-left: 5px;
  font-weight: bold;
}

/* Skills Section Styling */
.skills-section {
  margin-top: 15px;
  border-top: 1px solid #eee;
  padding-top: 10px;
}

.skills-section h4 {
  margin: 0 0 10px 0;
  font-size: 0.9em;
  color: #555;
  text-transform: uppercase;
  text-align: center;
}

.skills-container {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
  justify-content: center;
}

.skill-chip {
  padding: 4px 8px;
  border: 1px solid #ddd;
  border-radius: 12px;
  font-size: 0.8em;
  color: #666;
  background-color: #f5f5f5;
  cursor: pointer;
  transition: all 0.2s;
  user-select: none;
}

.skill-chip:hover {
  background-color: #e0e0e0;
}

.skill-chip.selected {
  background-color: #4CAF50;
  color: white;
  border-color: #4CAF50;
  font-weight: bold;
  box-shadow: 0 1px 3px rgba(0,0,0,0.2);
}

</style>
