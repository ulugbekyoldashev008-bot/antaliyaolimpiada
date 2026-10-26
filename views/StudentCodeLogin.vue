<template>
  <div class="student-entry-page xz-entry">
    <section class="xz-entry-grid">
      <div class="xz-entry-copy">
        <span class="xz-kicker"><b>★</b> Antalya · Online Olympiad</span>
        <h1 class="xz-title">Antalya<span>Olympiad</span></h1>

        <div class="xz-stage-login">
          <div v-tilt="6" class="student-card entry-card xz-login-card">
            <div class="xz-card-badges">
              <span>STUDENT LOGIN</span>
              <span>★ OLYMPIAD</span>
            </div>

            <template v-if="mode === 'code'">
              <h2>Enter the test</h2>

              <form @submit.prevent="startExam" class="code-form entry-form">
                <label class="code-label">
                  Status code
                  <input
                    v-model="code"
                    maxlength="6"
                    minlength="6"
                    inputmode="numeric"
                    placeholder="582914"
                    required
                  />
                </label>
                <button class="primary-btn entry-start-btn" :disabled="loading">
                  {{ loading ? 'Checking...' : 'Start test' }}
                </button>
              </form>

            </template>

            <template v-else>
              <h2>Registration</h2>
              <p class="entry-subtitle">Enter your details, choose a subject and level. The test starts right away.</p>

              <form @submit.prevent="register" class="xz-register-form">
                <label>
                  Full name
                  <input v-model.trim="reg.full_name" autocomplete="name" placeholder="For example: John Smith" required />
                </label>
                <label>
                  Learning center
                  <input v-model.trim="reg.center" list="xz-center-list" placeholder="Enter learning center name" required />
                  <datalist id="xz-center-list">
                    <option v-for="name in catalog.centers" :key="name" :value="name" />
                  </datalist>
                </label>
                <label class="xz-register-age">
                  Age
                  <input v-model.number="reg.age" type="number" min="4" max="99" inputmode="numeric" placeholder="12" required />
                </label>
                <div class="xz-register-row">
                  <label>
                    Subject
                    <select v-model="reg.subject" required @change="reg.level = ''">
                      <option value="" disabled>Choose a subject</option>
                      <option v-for="subject in catalog.subjects" :key="subject.id" :value="subject.id">{{ subject.name }}</option>
                    </select>
                  </label>
                  <label>
                    Level
                    <select v-model="reg.level" required :disabled="!selectedSubject">
                      <option value="" disabled>{{ selectedSubject ? 'Choose a level' : 'Choose a subject first' }}</option>
                      <option v-for="level in selectedSubject?.levels || []" :key="level.id" :value="level.id">{{ level.name }}</option>
                    </select>
                  </label>
                </div>
                <button class="primary-btn entry-start-btn" :disabled="loading || catalogLoading">
                  {{ loading ? 'Preparing test...' : 'Enter the test' }}
                </button>
              </form>

              <button type="button" class="xz-register-toggle is-back" @click="closeRegister">
                ← Log in with status code
              </button>
            </template>

            <p v-if="error" class="error-box">{{ error }}</p>

            <RouterLink to="/admin/login" class="admin-login-link entry-admin-link">
              Admin panel login
            </RouterLink>
          </div>
        </div>
      </div>

      <div class="xz-stage">
        <AntalyaScene />
        <div class="xz-orbit-labels">
          <span>KALEİÇİ</span>
          <span>YİVLİ MİNARE</span>
          <span>HIDIRLIK KULESİ</span>
        </div>
        <div class="xz-subjects xz-stage-subjects">
          <span>🧮 Mathematics</span>
          <span>💻 IT</span>
          <span>🖥️ Computer</span>
          <span>🇬🇧 English</span>
        </div>

      </div>
    </section>

    <div v-if="showResultModal" class="modal-backdrop result-lookup-backdrop" @click.self="closeResultModal">
      <div class="answer-modal result-lookup-modal">
        <button type="button" class="modal-close-btn" @click="closeResultModal">×</button>

        <template v-if="!publicResult">
          <div class="result-modal-icon">📊</div>
          <h2>View result</h2>
          <p>Enter the 6-digit status code you were given to view your result.</p>

          <form @submit.prevent="lookupResult" class="result-lookup-form">
            <label class="code-label">
              Status code
              <input
                v-model="resultCode"
                maxlength="6"
                minlength="6"
                inputmode="numeric"
                placeholder="582914"
                required
              />
            </label>
            <button class="primary-btn" :disabled="resultLoading">
              {{ resultLoading ? 'Searching...' : 'View result' }}
            </button>
          </form>

          <p v-if="resultError" class="error-box">{{ resultError }}</p>
        </template>

        <template v-else>
          <div class="result-modal-icon success">✓</div>
          <h2>Test completed</h2>
          <p class="result-meta">
            {{ publicResult.student_full_name }} · {{ publicResult.subject_name }} · {{ publicResult.level_name }}
          </p>

          <div class="public-result-details">
            <div>
              <span>Status</span>
              <b>Result sent to the admin</b>
            </div>
            <div>
              <span>Learning center</span>
              <b>{{ publicResult.center_name }}</b>
            </div>
            <div>
              <span>Time spent</span>
              <b>{{ formatSeconds(publicResult.spent_seconds) }}</b>
            </div>
          </div>

          <button type="button" class="secondary-btn block-link" @click="resetResultLookup">
            Check another code
          </button>
        </template>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, reactive, ref } from 'vue'
import { useRouter } from 'vue-router'
import api from '../api/axios'
import AntalyaScene from '../components/AntalyaScene.vue'

const router = useRouter()
const code = ref('')
const loading = ref(false)
const error = ref('')
const showResultModal = ref(false)
const resultCode = ref('')
const resultLoading = ref(false)
const resultError = ref('')
const publicResult = ref(null)

// Ro'yxatdan o'tish: o'quvchi o'zi fan va darajani tanlab testga kiradi.
const mode = ref('code')
const catalog = reactive({ subjects: [], centers: [] })
const catalogLoading = ref(false)
const reg = reactive({ full_name: '', center: '', age: '', subject: '', level: '' })
const selectedSubject = computed(() => catalog.subjects.find(item => item.id === reg.subject))

function cleanCode(value) {
  return String(value || '').trim()
}

// Backend xabarlari o'zbekcha, o'quvchiga inglizchasi ko'rsatiladi.
const ERRORS_EN = {
  'Status code kiriting.': 'Please enter the status code.',
  'Status code yuborilmadi.': 'Please enter the status code.',
  'Bunday code topilmadi.': 'This code was not found.',
  'Bu code oldin ishlatilgan.': 'This code has already been used.',
  'Bu test allaqachon yakunlangan.': 'This test has already been completed.',
  'Bu fan va daraja uchun testlar hali qo‘shilmagan.': 'Tests for this subject and level have not been added yet.',
  'Bu o‘quvchi hali testni yakunlamagan.': 'This student has not finished the test yet.',
  'Bu code uchun natija topilmadi.': 'No result found for this code.',
  'Juda ko‘p noto‘g‘ri code kiritildi. 10 daqiqadan keyin qayta urinib ko‘ring.': 'Too many wrong codes. Please try again in 10 minutes.',
}

function englishError(e, fallback) {
  const detail = e.response?.data?.detail
  if (e.response?.status === 429) return ERRORS_EN['Juda ko‘p noto‘g‘ri code kiritildi. 10 daqiqadan keyin qayta urinib ko‘ring.']
  return ERRORS_EN[detail] || fallback
}

function formatSeconds(seconds) {
  const total = Number(seconds || 0)
  const min = Math.floor(total / 60)
  const sec = total % 60
  return `${min} min ${sec} sec`
}

function openResultModal() {
  showResultModal.value = true
  resultCode.value = cleanCode(code.value)
  resultError.value = ''
  publicResult.value = null
}

function closeResultModal() {
  showResultModal.value = false
}

function resetResultLookup() {
  publicResult.value = null
  resultCode.value = ''
  resultError.value = ''
}

async function openExam(finalCode) {
  const res = await api.post('/exam/start/', { code: finalCode })
  sessionStorage.setItem('exam_payload', JSON.stringify({ ...res.data, code: finalCode }))
  router.push('/exam')
}

async function startExam() {
  loading.value = true
  error.value = ''
  try {
    await openExam(cleanCode(code.value))
  } catch (e) {
    error.value = englishError(e, 'Invalid or already used code.')
  } finally {
    loading.value = false
  }
}

async function openRegister() {
  mode.value = 'register'
  error.value = ''
  if (catalog.subjects.length) return
  catalogLoading.value = true
  try {
    const res = await api.get('/exam/catalog/')
    catalog.subjects = res.data.subjects || []
    catalog.centers = res.data.centers || []
  } catch (e) {
    error.value = 'Could not load the subject list. Please refresh the page.'
  } finally {
    catalogLoading.value = false
  }
}

function closeRegister() {
  mode.value = 'code'
  error.value = ''
}

async function register() {
  loading.value = true
  error.value = ''
  try {
    const res = await api.post('/exam/register/', {
      full_name: reg.full_name,
      center: reg.center,
      age: reg.age,
      level: reg.level,
    })
    code.value = res.data.code
    await openExam(res.data.code)
  } catch (e) {
    error.value = englishError(e, 'Registration failed. Please try again.')
  } finally {
    loading.value = false
  }
}

async function lookupResult() {
  resultLoading.value = true
  resultError.value = ''
  publicResult.value = null
  try {
    const res = await api.post('/exam/result/', { code: cleanCode(resultCode.value) })
    publicResult.value = res.data
  } catch (e) {
    resultError.value = englishError(e, 'Result not found.')
  } finally {
    resultLoading.value = false
  }
}
</script>
