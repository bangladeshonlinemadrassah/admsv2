
  <script>
    // =========================
    // JavaScript কোড - Google Sheets সংযোগ সহ
    // =========================
    const API_URL = "https://script.google.com/macros/s/AKfycbwZnb6Lr-_IN1FzgvhQ-em6NjaqVIWWpmEKHinchUbY4Bt8NhOssIx_dRJ4MiSFmfKJ4g/exec";

    // DOM হেল্পার ফাংশন
    function $(sel, root = document) { return root.querySelector(sel); }
    function $all(sel, root = document) { return Array.from(root.querySelectorAll(sel)); }

    // পেজ স্টেট ম্যানেজমেন্ট
    let currentSubject = null;
    const loginPage = $('#login-page');
    const mainPage = $('#main-page');
    const subjectPage = $('#subject-page');
    const warningModal = $('#warning-modal');
    let allClasses = [];

    // সাবজেক্টের তালিকা (Dakhil-26 এর জন্য)
    const availableSubjects = [
      "কুরআন মাজীদ ও তাজভীদ", "হাদিস শরীফ", "আরবি প্রথম পত্র", "আরবি দ্বিতীয় পত্র",
      "আকাইদ ও ফিকহ", "বাংলা প্রথম পত্র", "বাংলা দ্বিতীয় পত্র", "ইংরেজি প্রথম পত্র",
      "ইংরেজি দ্বিতীয় পত্র", "গণিত", "ইসলামের ইতিহাস", "তথ্য ও যোগাযোগ প্রযুক্তি",
      "পদার্থবিজ্ঞান", "রসায়ন", "উচ্চতর গণিত", "জীববিজ্ঞান",
      "বাংলাদেশ ও বিশ্বপরিচয়", "কৃষি শিক্ষা",
      "মানতিক", "গার্হস্থ্য বিজ্ঞান", "পৌরনীতি ও নাগরিকতা", "মিশন",
      "ইক্সাম", "অন্যান্য"
    ];

    // localStorage keys
    const LOGIN_EXPIRY_KEY = 'dakhil26_login_expiry';
    const IS_LOGGED_IN_KEY = 'dakhil26_is_logged_in';

    // পেজ উপরে স্ক্রল করার ফাংশন
    function scrollToTop() {
      window.scrollTo({ top: 0, behavior: 'smooth' });
    }

    // লগইন expiry তারিখ সেট করা
    function setLoginExpiry() {
      const now = new Date();
      const oneMonthFromNow = new Date(now.setMonth(now.getMonth() + 1));
      localStorage.setItem(LOGIN_EXPIRY_KEY, oneMonthFromNow.getTime());
      localStorage.setItem(IS_LOGGED_IN_KEY, 'true');
    }

    // লগইন expiry চেক করা
    function checkLoginExpiry() {
      const expiryTime = localStorage.getItem(LOGIN_EXPIRY_KEY);
      const isLoggedIn = localStorage.getItem(IS_LOGGED_IN_KEY);
      
      if (!expiryTime || !isLoggedIn) {
        return false;
      }
      
      const now = new Date().getTime();
      return now < parseInt(expiryTime);
    }

    // লগআউট করা
    function logout() {
      localStorage.removeItem(LOGIN_EXPIRY_KEY);
      localStorage.removeItem(IS_LOGGED_IN_KEY);
      loginPage.style.display = 'flex';
      mainPage.style.display = 'none';
      subjectPage.style.display = 'none';
    }

    // ওয়ার্নিং মোডাল দেখানোর ফাংশন
    function showWarningModal() {
      warningModal.classList.add('show');
    }

    // ওয়ার্নিং মোডাল বন্ধ করার ফাংশন
    function closeWarningModal() {
      warningModal.classList.remove('show');
    }

    // নোটিফিকেশন দেখানোর ফাংশন
    function showNotification(message, isError = false, isWarning = false) {
      const notification = $('#notification');
      notification.innerHTML = `<i class="${isError ? 'fas fa-exclamation-circle' : isWarning ? 'fas fa-exclamation-triangle' : 'fas fa-check-circle'}"></i> ${message}`;
      notification.className = 'notification ' + (isError ? 'error' : isWarning ? 'warning' : 'success');
      notification.classList.add('show');

      setTimeout(() => {
        notification.classList.remove('show');
      }, 3000);
    }

    // পাসওয়ার্ড চেক করার ফাংশন
    function checkPassword() {
      const password = document.getElementById('password').value;
      const errorElement = document.getElementById('error-message');
      
      if (password === 'USERDAKHIL26') {
        loginPage.style.display = 'none';
        mainPage.style.display = 'block';
        setLoginExpiry();
        showWarningModal();
        loadInitialData();
        scrollToTop();
      } else {
        errorElement.style.display = 'block';
        setTimeout(() => {
          errorElement.style.display = 'none';
        }, 3000);
      }
    }

    // Enter কী চাপলে check password
    document.getElementById('password').addEventListener('keypress', function(e) {
      if (e.key === 'Enter') {
        checkPassword();
      }
    });

    // টাইমস্ট্যাম্প থেকে তারিখ বের করা
    function getDateFromTimestamp(timestamp) {
      if (!timestamp) return getTodayDate();

      try {
        if (typeof timestamp === 'string') {
          const date = new Date(timestamp);
          if (!isNaN(date)) {
            const year = date.getFullYear();
            const month = String(date.getMonth() + 1).padStart(2, '0');
            const day = String(date.getDate()).padStart(2, '0');
            return `${year}-${month}-${day}`;
          }
        }
        return getTodayDate();
      } catch (error) {
        console.error("Timestamp parsing error:", error, timestamp);
        return getTodayDate();
      }
    }

    // টাইমস্ট্যাম্প থেকে সময় বের করা
    function getTimeFromTimestamp(timestamp) {
      if (!timestamp) return '3:37 PM';

      try {
        const date = new Date(timestamp);
        if (!isNaN(date)) {
          const hours = date.getHours();
          const minutes = String(date.getMinutes()).padStart(2, '0');
          const period = hours >= 12 ? 'PM' : 'AM';
          const hour12 = hours % 12 || 12;
          return `${hour12}:${minutes} ${period}`;
        }
        return '3:37 PM';
      } catch (error) {
        console.error("Time parsing error:", error, timestamp);
        return '3:37 PM';
      }
    }

    // আজকের তারিখ ফরম্যাট করা
    function getTodayDate() {
      const today = new Date();
      const year = today.getFullYear();
      const month = String(today.getMonth() + 1).padStart(2, '0');
      const day = String(today.getDate()).padStart(2, '0');
      return `${year}-${month}-${day}`;
    }

    // বাংলা মাসের নাম
    const banglaMonths = [
      'জানুয়ারী', 'ফেব্রুয়ারী', 'মার্চ', 'এপ্রিল', 'মে', 'জুন',
      'জুলাই', 'আগস্ট', 'সেপ্টেম্বর', 'অক্টোবর', 'নভেম্বর', 'ডিসেম্বর'
    ];

    // তারিখ ফরম্যাট করা (বাংলা ফরম্যাটে)
    function formatDateToBangla(dateStr) {
      if (!dateStr) return '';

      try {
        if (/^\d{4}-\d{2}-\d{2}$/.test(dateStr)) {
          const [year, month, day] = dateStr.split('-');
          return `${parseInt(day)} ${banglaMonths[parseInt(month) - 1]}, ${year}`;
        }
        return dateStr;
      } catch (error) {
        console.error("Date formatting error:", error, dateStr);
        return dateStr;
      }
    }

    // সাবজেক্ট ম্যাচিং ফাংশন
    function findBestMatchingSubject(subjectFromSheet) {
      if (!subjectFromSheet) return null;

      const cleanSubject = subjectFromSheet.toString().trim();
      for (const availableSubject of availableSubjects) {
        if (cleanSubject === availableSubject) {
          return availableSubject;
        }
      }
      return cleanSubject;
    }

    // ডেটা ফেচ করা
    async function fetchData() {
      try {
        showNotification("ডেটা লোড হচ্ছে...", false);
        const res = await fetch(API_URL, {
          method: 'GET',
          headers: {
            'Accept': 'application/json'
          }
        });

        if (!res.ok) {
          throw new Error(`HTTP error! status: ${res.status}`);
        }

        const data = await res.json();
        if (data && data.ok) {
          showNotification("ডেটা সফলভাবে লোড হয়েছে!");
          console.log("API Response:", data);
          return data.rows || [];
        } else {
          throw new Error(data.error || "ডেটা লোড করতে সমস্যা");
        }
      } catch (error) {
        console.error("ডেটা লোড করতে সমস্যা:", error);
        showNotification("ডেটা লোড করা যায়নি: " + error.message, true);
        return [];
      }
    }

    // ডেটা নরমালাইজ করা
    function normalizeRow(row, index) {
      try {
        const timestamp = row.timestamp || '';
        const subjectFromSheet = row.subject || 'Unknown Subject';
        const link = row.link || '#';
        const title = row.title || 'ক্লাস ভিডিও';
        const date = getDateFromTimestamp(timestamp);
        const time = getTimeFromTimestamp(timestamp);
        const matchedSubject = findBestMatchingSubject(subjectFromSheet) || subjectFromSheet;

        return {
          subject: matchedSubject,
          originalSubject: subjectFromSheet,
          link: link,
          title: title,
          date: date,
          time: time,
          serial: index + 1,
          timestamp: timestamp
        };
      } catch (error) {
        console.error("Row normalization error:", error, row);
        return null;
      }
    }

    // Check if exam link is expired (10 hours = 36,000,000 milliseconds)
    function isExamLinkExpired(timestamp) {
      if (!timestamp) return false;
      try {
        const examTime = new Date(timestamp).getTime();
        const now = new Date().getTime();
        const tenHoursInMs = 10 * 60 * 60 * 1000; // 10 hours in milliseconds
        return (now - examTime) > tenHoursInMs;
      } catch (error) {
        console.error("Exam expiry check error:", error, timestamp);
        return false;
      }
    }

    // কার্ড রেন্ডার করা
    function renderCards(rows, root) {
      if (!rows || rows.length === 0) {
        root.innerHTML = '<div class="empty-state"><svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" /></svg><p class="bangla-font">কোন ক্লাস পাওয়া যায়নি</p></div>';
        return;
      }

      root.innerHTML = '';
      rows.forEach(r => {
        if (!r) return;

        let buttonText;
        let isDisabled = false;
        let link = r.link;
        if (r.subject === "মিশন") {
          buttonText = "মিশন সম্পন্ন করুন";
        } else if (r.subject === "ইক্সাম") {
          if (isExamLinkExpired(r.timestamp)) {
            buttonText = "পরীক্ষায় অংশগ্রহণ করার সময় শেষ হয়ে গেছে";
            isDisabled = true;
            link = "#";
          } else {
            buttonText = "পরীক্ষায় অংশগ্রহণ করুন";
          }
        } else {
          buttonText = "ক্লাস দেখুন";
        }

        const card = document.createElement('div');
        card.className = 'card fade-in';
        card.innerHTML = `
          <span class="card-badge">#${r.serial}</span>
          <div class="subject"><i class="fas fa-book"></i> <span class="bangla-font">${r.subject}</span></div>
          <div class="video-title bangla-font">${r.title}</div>
          <div class="time"><i class="fas fa-clock"></i> ${r.time} | <span class="bangla-font">${formatDateToBangla(r.date)}</span></div>
          <a class="btn${isDisabled ? ' disabled' : ''}" ${isDisabled ? '' : 'target="_blank" rel="noopener"'} href="${link}"><i class="fas fa-play-circle"></i> <span class="bangla-font">${buttonText}</span></a>
        `;
        root.appendChild(card);
      });
    }

    // আজকের ক্লাস ফিল্টার করা
    function filterTodayClasses(classes) {
      const today = getTodayDate();
      return classes.filter(cls => cls && cls.date === today);
    }

    // আজকের ক্লাসের সারাংশ দেখানো
    function renderTodayClassesSummary(classes) {
      const today = getTodayDate();
      const todayClasses = filterTodayClasses(classes);
      const container = $('#today-classes-body');

      if (!todayClasses || todayClasses.length === 0) {
        container.innerHTML = `<tr><td colspan="3" class="bangla-font" style="text-align: center; padding: 20px;">আজকের জন্য কোন ক্লাস পাওয়া যায়নি</td></tr>`;
        return;
      }

      let tableHTML = '';
      todayClasses.forEach(cls => {
        if (!cls) return;
        tableHTML += `
          <tr>
            <td><span class="subject-badge bangla-font">${cls.subject}</span></td>
            <td class="class-info bangla-font">${cls.title}</td>
            <td class="bangla-font">${cls.time}</td>
          </tr>
        `;
      });
      container.innerHTML = tableHTML;
    }

    // সময় অনুযায়ী সাজানো
    function sortByTime(rows, order = 'desc') {
      if (!rows) return [];
      return rows.filter(r => r).slice().sort((a, b) => {
        const timeA = new Date(a.timestamp).getTime();
        const timeB = new Date(b.timestamp).getTime();
        return order === 'asc' ? timeA - timeB : timeB - timeA;
      });
    }

    // সাবজেক্ট পৃষ্ঠা দেখানো
    function showSubjectPage(subject) {
      currentSubject = subject;
      $('#subject-title').innerHTML = `<i class="fas fa-book"></i> <span class="bangla-font">${subject}</span>`;
      mainPage.style.display = 'none';
      subjectPage.style.display = 'block';
      scrollToTop();
      loadSubjectData(subject);
    }

    // মেইন পৃষ্ঠায় ফিরে যাওয়া
    function showMainPage() {
      subjectPage.style.display = 'none';
      mainPage.style.display = 'block';
      currentSubject = null;
      scrollToTop();
    }

    // সাবজেক্ট ডেটা লোড করা
    function loadSubjectData(subject) {
      const grid = $('#classGrid');
      const searchInput = $('#search');
      const sortSelect = $('#sort');

      grid.innerHTML = '<div class="card"><div class="subject bangla-font">লোড হচ্ছে... <span class="loading"></span></div></div>';

      let rows = allClasses.filter(r => r && String(r.subject).trim() === subject);
      rows = sortByTime(rows, 'desc'); // Default to newest first
      renderCards(rows, grid);

      searchInput.addEventListener('input', () => {
        const q = searchInput.value.toLowerCase();
        const filtered = rows.filter(r =>
          (r.subject || '').toLowerCase().includes(q) ||
          (r.title || '').toLowerCase().includes(q)
        );
        renderCards(filtered, grid);
      });

      sortSelect.addEventListener('change', () => {
        const order = sortSelect.value;
        const sorted = sortByTime(rows, order);
        renderCards(sorted, grid);
      });
    }

    // ইভেন্ট লিসেনার সেটআপ
    function setupEventListeners() {
      $all('.subject-link').forEach(link => {
        link.addEventListener('click', (e) => {
          e.preventDefault();
          const subject = link.getAttribute('data-name');
          showSubjectPage(subject);
        });
      });

      $('.back-button').addEventListener('click', (e) => {
        e.preventDefault();
        showMainPage();
      });

      const searchInput = $('#search-subjects');
      if (searchInput) {
        searchInput.addEventListener('input', () => {
          const q = searchInput.value.trim().toLowerCase();
          $all('.pill').forEach(p => {
            const name = p.dataset.name.toLowerCase();
            p.style.display = name.includes(q) ? '' : 'none';
          });
        });
      }

      $('#mission-stats').addEventListener('click', () => {
        showSubjectPage('মিশন');
      });

      $('#exam-stats').addEventListener('click', () => {
        showSubjectPage('ইক্সাম');
      });
    }

    // প্রাথমিক ডেটা লোড
    async function loadInitialData() {
      try {
        const data = await fetchData();
        allClasses = data.map((row, index) => normalizeRow(row, index)).filter(r => r);
        console.log("Processed classes:", allClasses);
        renderTodayClassesSummary(allClasses);
        setupEventListeners();
      } catch (error) {
        console.error("Initial data loading error:", error);
        showNotification("ডেটা লোড করতে সমস্যা হয়েছে", true);
      }
    }

    // পেজ লোড হলে ইনিশিয়ালাইজেশন
    document.addEventListener('DOMContentLoaded', () => {
      const isLoggedIn = checkLoginExpiry();
      
      if (isLoggedIn) {
        loginPage.style.display = 'none';
        mainPage.style.display = 'block';
        loadInitialData();
      } else {
        loginPage.style.display = 'flex';
        mainPage.style.display = 'none';
        subjectPage.style.display = 'none';
        localStorage.removeItem(LOGIN_EXPIRY_KEY);
        localStorage.removeItem(IS_LOGGED_IN_KEY);
      }

      scrollToTop();

      setTimeout(() => {
        document.querySelectorAll('.fade-in').forEach(el => {
          el.style.opacity = '1';
          el.style.transform = 'translateY(0)';
        });
      }, 100);
    });
  </script>
</body>
</html>
