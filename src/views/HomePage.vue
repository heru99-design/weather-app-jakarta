<template>
  <ion-page>
    <ion-header>
      <ion-toolbar color="primary" class="custom-header">
        <div class="header-content">
          <ion-title class="ion-text-center main-title">Weather Jakarta</ion-title>
          <div class="ion-text-center subtitle">Prakiraan Cuaca Per Jam - Open-Meteo API</div>
          <div class="datetime-inline">
            <span>{{ formattedDateTime }}</span>
          </div>
        </div>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      
      <div class="header-controls">
        <ion-button expand="block" @click="fetchWeatherData" :disabled="isLoading">
          <ion-icon :icon="refresh" slot="start"></ion-icon>
          {{ isLoading ? 'Memuat...' : 'Refresh Data' }}
        </ion-button>
        
        <div class="sync-status-only">
          <small>{{ syncStatus }}</small>
        </div>
      </div>

      <!-- Loading State -->
      <div v-if="isLoading && weatherData.length === 0" class="ion-text-center ion-padding">
        <ion-spinner name="crescent"></ion-spinner>
        <p>Memuat data cuaca Jakarta...</p>
      </div>

      <!-- Error State -->
      <ion-card v-else-if="errorMessage" color="danger">
        <ion-card-header>
          <ion-card-title>Gagal Memuat Data</ion-card-title>
        </ion-card-header>
        <ion-card-content>
          {{ errorMessage }}
        </ion-card-content>
      </ion-card>

      <!-- Data List -->
      <ion-list v-else>
        <ion-item-divider color="light">
          <ion-label>
            <h2 class="ion-text-center">Prakiraan Suhu Jakarta (10 Jam ke Depan)</h2>
          </ion-label>
        </ion-item-divider>

        <ion-item v-for="(item, index) in weatherData" :key="index" class="weather-item">
          <ion-icon :icon="timeIcon" slot="start" color="primary" class="time-icon"></ion-icon>
          
          <ion-label>
            <h2 class="time-text">{{ formatTime(item.time) }}</h2>
            <p class="date-text">{{ formatDate(item.time) }}</p>
          </ion-label>
          
          <div slot="end" class="temp-tag">
            <ion-icon :icon="thermometerIcon" :color="getTempColor(item.temperature)"></ion-icon>
            <h3 class="temp-text">{{ item.temperature.toFixed(1) }}°C</h3>
          </div>
        </ion-item>
      </ion-list>

    </ion-content>

    <!-- Footer Fixed -->
    <ion-footer>
      <ion-toolbar class="footer-toolbar">
        <div class="footer-text">
          <p>Data dari Open-Meteo API (10 data pertama dari jam sekarang)</p>
          <a href="https://open-meteo.com/" target="_blank">https://open-meteo.com/</a>
        </div>
      </ion-toolbar>
    </ion-footer>

  </ion-page>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import {
  IonPage, IonHeader, IonFooter, IonToolbar, IonTitle, IonContent,
  IonButton, IonIcon, IonSpinner, IonCard, IonCardHeader,
  IonCardTitle, IonCardContent, IonList, IonItem, IonLabel,
  IonItemDivider
} from '@ionic/vue';
import { refresh, time as timeIcon, thermometer as thermometerIcon } from 'ionicons/icons';

// Interface untuk data cuaca
interface WeatherData {
  time: string;
  temperature: number;
}

interface ApiResponse {
  hourly: {
    time: string[];
    temperature_2m: number[];
  };
}

// Reactive state
const weatherData = ref<WeatherData[]>([]);
const isLoading = ref<boolean>(false);
const errorMessage = ref<string>('');
const currentTime = ref<Date>(new Date());
const syncStatus = ref<string>('⏱️ Sinkronisasi: Menunggu...');

// Format tanggal dan waktu real-time
const formattedDateTime = computed(() => {
  const now = currentTime.value;
  const dateStr = now.toLocaleDateString('id-ID', { 
    weekday: 'long', year: 'numeric', month: 'long', day: 'numeric',
    timeZone: 'Asia/Jakarta' 
  });
  const timeStr = now.toLocaleTimeString('id-ID', { 
    hour: '2-digit', minute: '2-digit', second: '2-digit', hour12: false,
    timeZone: 'Asia/Jakarta' 
  });
  return `${dateStr} | ${timeStr} WIB`;
});

// Format waktu dari API
const formatTime = (timeString: string): string => {
  const date = new Date(timeString);
  return date.toLocaleTimeString('id-ID', { 
    hour: '2-digit', 
    minute: '2-digit',
    hour12: false,
    timeZone: 'Asia/Jakarta'
  }) + ' WIB';
};

// Format tanggal dari API
const formatDate = (timeString: string): string => {
  const date = new Date(timeString);
  return date.toLocaleDateString('id-ID', { 
    weekday: 'long',
    day: 'numeric',
    month: 'long',
    year: 'numeric',
    timeZone: 'Asia/Jakarta'
  });
};

// Warna berdasarkan suhu
const getTempColor = (temp: number): string => {
  if (temp >= 32) return 'danger';
  if (temp >= 28) return 'warning';
  if (temp >= 24) return 'primary';
  return 'success';
};

// Fetch data dari API Open-Meteo
const fetchWeatherData = async () => {
  isLoading.value = true;
  errorMessage.value = '';
  syncStatus.value = '⏱️ Sinkronisasi: Mengambil data...';

  try {
    const apiUrl = 'https://api.open-meteo.com/v1/forecast?latitude=-6.2&longitude=106.8&hourly=temperature_2m';
    
    const response = await fetch(apiUrl);
    if (!response.ok) throw new Error(`HTTP Error: ${response.status}`);
    
    const data: ApiResponse = await response.json();
    
    // Gabungkan time dan temperature_2m menjadi array objek
    const combinedData: WeatherData[] = data.hourly.time.map((time, index) => ({
      time: time,
      temperature: data.hourly.temperature_2m[index]
    }));
    
    // LOGIKA BARU: Mulai dari jam berikutnya yang utuh
    const now = new Date();
    const currentHour = now.getHours();
    const startHour = currentHour + 1;
    
    // Cari index data yang jam-nya >= startHour
    const startIndex = combinedData.findIndex(item => {
      const itemDate = new Date(item.time);
      const itemHour = itemDate.getHours();
      
      return itemHour >= startHour;
    });
    
    // Jika tidak ada data yang cocok, ambil dari awal
    const finalStartIndex = startIndex === -1 ? 0 : startIndex;
    
    // Ambil 10 data mulai dari jam berikutnya
    weatherData.value = combinedData.slice(finalStartIndex, finalStartIndex + 10);
    
    syncStatus.value = `✅ Data berhasil dimuat (10 jam mulai ${startHour.toString().padStart(2, '0')}:00 WIB)`;
    
  } catch (error: any) {
    errorMessage.value = error.message || 'Gagal memuat data cuaca';
    syncStatus.value = '❌ Gagal sinkronisasi';
  } finally {
    isLoading.value = false;
  }
};

onMounted(() => {
  fetchWeatherData();
  
  // Update waktu setiap detik
  setInterval(() => {
    currentTime.value = new Date();
  }, 1000);
});
</script>

<style scoped>
.custom-header {
  padding: 0;
}

.header-content {
  padding: 12px 16px 8px;
}

.main-title {
  font-size: 1.4rem !important;
  margin-bottom: 4px !important;
  padding: 0 !important;
}

.subtitle {
  font-size: 0.75rem;
  color: rgba(255,255,255,0.9);
  margin-bottom: 6px;
}

.datetime-inline {
  font-size: 0.7rem;
  color: rgba(255,255,255,0.85);
  text-align: center;
  padding-top: 4px;
  border-top: 1px solid rgba(255,255,255,0.2);
}

.header-controls {
  margin-bottom: 15px;
}

.sync-status-only {
  text-align: center;
  margin-top: 8px;
}

.sync-status-only small {
  font-size: 0.7rem;
  color: #666;
}

.weather-item {
  --padding-start: 12px;
  --padding-end: 12px;
  --min-height: 60px;
}

.time-icon {
  font-size: 1.5rem;
  margin-right: 12px;
}

.time-text {
  font-weight: 600;
  font-size: 1rem;
  color: #333;
  margin: 0;
}

.date-text {
  font-size: 0.75rem;
  color: #666;
  margin: 2px 0 0 0;
}

.temp-tag {
  display: flex;
  align-items: center;
  gap: 6px;
}

.temp-text {
  font-size: 1.1rem;
  font-weight: 700;
  margin: 0;
  min-width: 70px;
  text-align: right;
}

/* Footer fixed */
ion-footer {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 100;
}

.footer-toolbar {
  --background: #f5f5f5;
  --color: #888;
  border-top: 1px solid #e0e0e0;
}

.footer-text {
  text-align: center;
  padding: 10px 16px;
  font-size: 0.7rem;
  color: #888;
  width: 100%;
}

.footer-text p {
  margin: 0 0 4px 0;
}

.footer-text a {
  color: #3880ff;
  text-decoration: none;
}

/* Padding bawah agar item terakhir tidak tertutup footer */
ion-content {
  --padding-bottom: 100px !important;
}
</style>