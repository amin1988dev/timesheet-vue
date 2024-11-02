<template>
  <div class="calendar-wrapper">
    <div class="calendar-container">
      <div class="d-flex justify-content-between align-items-center mb-3">
        <button class="btn btn-outline-secondary" @click="prevMonth">← Mese precedente</button>
        <h3>{{ formattedCurrentMonth }}</h3>
        <button class="btn btn-outline-secondary" @click="nextMonth">Mese successivo →</button>
      </div>
      <div class="calendar-grid">
        <div v-for="day in daysOfWeek" :key="day" class="calendar-day-header">
          {{ day }}
        </div>
        <div v-for="(date, index) in dates" :key="index" class="calendar-cell">
          <button
            v-if="date"
            :class="[ 
              'btn',
              'calendar-btn',
              isToday(date) ? 'btn-warning' : (isPrevMonth(date) ? 'btn-prev-month' : isNextMonth(date) ? 'btn-next-month' : 'btn-default')
            ]"
            @click="openModal(date)"
          >
            {{ date }}
            <span v-if="isDateFilled(date)" class="check-mark">✔</span>
          </button>
          <div v-else class="empty-cell"></div>
        </div>
      </div>

      <!-- Modal -->
      <div v-if="selectedDate" class="modal show d-block" tabindex="-1">
        <div class="modal-dialog">
          <div class="modal-content">
            <div class="modal-header">
              <h5 class="modal-title">Inserisci le ore per {{ selectedDate }}</h5>
              <button type="button" class="btn-close" @click="closeModal"></button>
            </div>
            <div class="modal-body">
              <input
                v-model.number="hours"
                type="number"
                class="form-control"
                placeholder="Ore lavorate"
              />
            </div>
            <div class="modal-footer">
              <button class="btn btn-success" @click="saveHours">Ok</button>
              <button class="btn btn-secondary" @click="closeModal">Cancella</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      currentMonth: new Date().getMonth(),
      currentYear: new Date().getFullYear(),
      dates: [],
      selectedDate: null,
      hours: '',
      daysOfWeek: ['Lun', 'Mar', 'Mer', 'Gio', 'Ven', 'Sab', 'Dom'],
      filledDates: [],
    };
  },
  computed: {
    formattedCurrentMonth() {
      const monthNames = [
        'Gennaio', 'Febbraio', 'Marzo', 'Aprile', 'Maggio', 'Giugno',
        'Luglio', 'Agosto', 'Settembre', 'Ottobre', 'Novembre', 'Dicembre'
      ];
      return `${monthNames[this.currentMonth]} ${this.currentYear}`;
    },
  },
  methods: {
    generateCalendar() {
      const daysInMonth = this.getDaysInMonth(this.currentMonth, this.currentYear);
      const firstDayOfMonth = new Date(this.currentYear, this.currentMonth, 1);
      const datesArray = [];
      const firstDayIndex = (firstDayOfMonth.getDay() + 6) % 7;
      for (let i = 0; i < firstDayIndex; i++) {
        datesArray.push(null);
      }
      for (let day = 1; day <= daysInMonth; day++) {
        datesArray.push(day);
      }
      this.dates = datesArray;
    },
    getDaysInMonth(month, year) {
      if (month === 1) {
        return (year % 4 === 0 && (year % 100 !== 0 || year % 400 === 0)) ? 29 : 28;
      }
      return [0, 2, 4, 6, 7, 9, 11].includes(month) ? 31 : 30;
    },
    isToday(date) {
      const today = new Date();
      const checkDate = new Date(this.currentYear, this.currentMonth, date);
      return (
        today.getFullYear() === checkDate.getFullYear() &&
        today.getMonth() === checkDate.getMonth() &&
        today.getDate() === checkDate.getDate()
      );
    },
    isPrevMonth(date) {
      const today = new Date(this.currentYear, this.currentMonth, date);
      return today < new Date(this.currentYear, this.currentMonth, 1);
    },
    isNextMonth(date) {
      const today = new Date(this.currentYear, this.currentMonth, date);
      return today > new Date(this.currentYear, this.currentMonth + 1, 0);
    },
    isDateFilled(date) {
      const formattedDate = `${this.currentYear}-${String(this.currentMonth + 1).padStart(2, '0')}-${String(date).padStart(2, '0')}`;
      const entry = this.filledDates.find(entry => entry.date === formattedDate);
      return entry && entry.hours > 0; // Mostra la spunta solo se le ore sono maggiori di 0
    },
    async loadFilledDates() {
      try {
        const response = await axios.get('http://127.0.0.1:8000/api/timesheets');
        this.filledDates = response.data;
      } catch (error) {
        console.error('Errore nel caricamento delle date', error);
      }
    },
    openModal(date) {
      if (date) {
        this.selectedDate = `${this.currentYear}-${String(this.currentMonth + 1).padStart(2, '0')}-${String(date).padStart(2, '0')}`;
        const existingEntry = this.filledDates.find(entry => entry.date === this.selectedDate);
        this.hours = existingEntry && existingEntry.hours > 0 ? existingEntry.hours : ''; // Mostra vuoto se le ore sono 0 o non presenti
      }
    },
    closeModal() {
      this.selectedDate = null;
    },
    async saveHours() {
      try {
        const existingEntry = this.filledDates.find(entry => entry.date === this.selectedDate);
        if (existingEntry) {
          await axios.put(`http://127.0.0.1:8000/api/timesheets/${existingEntry.id}`, {
            date: this.selectedDate,
            hours: this.hours,
          });
          //alert('Ore aggiornate con successo!');
        } else {
          await axios.post('http://127.0.0.1:8000/api/timesheets', {
            date: this.selectedDate,
            hours: this.hours,
          });
          //alert('Ore salvate con successo!');
        }
        this.loadFilledDates();
        this.closeModal();
      } catch (error) {
        alert('Errore durante il salvataggio/aggiornamento');
      }
    },
    nextMonth() {
      if (this.currentMonth === 11) {
        this.currentMonth = 0;
        this.currentYear++;
      } else {
        this.currentMonth++;
      }
      this.generateCalendar();
      this.loadFilledDates();
    },
    prevMonth() {
      if (this.currentMonth === 0) {
        this.currentMonth = 11;
        this.currentYear--;
      } else {
        this.currentMonth--;
      }
      this.generateCalendar();
      this.loadFilledDates();
    },
  },
  mounted() {
    this.generateCalendar();
    this.loadFilledDates();
  },
};
</script>

<style scoped>
.calendar-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  padding-top: 50px;
}

.calendar-container {
  background-color: #42b883;
  padding: 20px;
  border-radius: 15px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 1100px;
  margin: auto;
}

.calendar-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 5px;
}

.calendar-day-header {
  font-weight: bold;
  text-align: center;
  padding: 10px 0;
  background-color: #f0f0f0;
  border-radius: 5px;
}

.calendar-cell {
  text-align: center;
  position: relative;
}

.calendar-btn {
  width: 100%;
  height: 80px;
  font-size: 1.2rem;
  border-radius: 0;
}

.btn-default {
  background-color: #fff;
  border: 1px solid #000;
  color: #000;
}

.check-mark {
  color: green;
  font-size: 1.5rem;
  position: absolute;
  top: 5px;
  right: 5px;
}

.empty-cell {
  height: 80px;
  visibility: hidden;
}

.btn-warning {
  background-color: #ffc107;
  color: #000;
}

.btn-outline-secondary {
  background-color: #35495e;
  color: #ffffff;
  border: 1px solid #35495e;
}

.btn-outline-secondary:hover {
  background-color: #2c3e50;
  border-color: #2c3e50;
}
</style>
