using System;
using System.Collections.Generic;
using System.Linq;

namespace SF2022UserNN.Lib
{
    public class Calculations
    {
        public string[] AvailablePeriods(
            TimeSpan[] startTimes,
            int[] durations,
            TimeSpan beginWorkingTime,
            TimeSpan endWorkingTime,
            int consultationTime)
        {
            // ВАЛИДАЦИЯ
            if (consultationTime <= 0)
                return new string[0];
            
            if (startTimes == null || durations == null)
                return new string[0];
            
            if (startTimes.Length != durations.Length)
                return new string[0];
            
            // ПРОВЕРКА: если нет консультаций, возвращаем весь день, разбитый на слоты
            if (startTimes.Length == 0)
            {
                var result = new List<string>();
                int begin = (int)beginWorkingTime.TotalMinutes;
                int end = (int)endWorkingTime.TotalMinutes;
                
                for (int start = begin; start + consultationTime <= end; start += consultationTime)
                {
                    int endTime = start + consultationTime;
                    result.Add($"{start / 60:D2}:{start % 60:D2}-{endTime / 60:D2}:{endTime % 60:D2}");
                }
                return result.ToArray();
            }
            
            // КОНВЕРТАЦИЯ В МИНУТЫ
            int beginMinutes = (int)beginWorkingTime.TotalMinutes;
            int endMinutes = (int)endWorkingTime.TotalMinutes;
            
            // СОЗДАЕМ СПИСОК ЗАНЯТЫХ ИНТЕРВАЛОВ
            var busySlots = new List<(int start, int end)>();
            
            for (int i = 0; i < startTimes.Length; i++)
            {
                int start = (int)startTimes[i].TotalMinutes;
                int end = start + durations[i];
                
                // Обрезаем по рабочему дню
                if (end > beginMinutes && start < endMinutes)
                {
                    busySlots.Add((Math.Max(start, beginMinutes), Math.Min(end, endMinutes)));
                }
            }
            
            // СОРТИРУЕМ
            busySlots = busySlots.OrderBy(x => x.start).ToList();
            
            // ОБЪЕДИНЯЕМ ПЕРЕСЕКАЮЩИЕСЯ ИНТЕРВАЛЫ
            var merged = new List<(int start, int end)>();
            foreach (var slot in busySlots)
            {
                if (merged.Count == 0 || slot.start > merged[merged.Count - 1].end)
                {
                    merged.Add(slot);
                }
                else
                {
                    var last = merged[merged.Count - 1];
                    merged[merged.Count - 1] = (last.start, Math.Max(last.end, slot.end));
                }
            }
            
            // НАХОДИМ СВОБОДНЫЕ ИНТЕРВАЛЫ
            var freeSlots = new List<(int start, int end)>();
            int current = beginMinutes;
            
            foreach (var busy in merged)
            {
                if (current < busy.start)
                {
                    freeSlots.Add((current, busy.start));
                }
                current = Math.Max(current, busy.end);
            }
            
            if (current < endMinutes)
            {
                freeSlots.Add((current, endMinutes));
            }
            
            // ФОРМИРУЕМ РЕЗУЛЬТАТ
            var result = new List<string>();
            foreach (var free in freeSlots)
            {
                for (int slotStart = free.start; slotStart + consultationTime <= free.end; slotStart += consultationTime)
                {
                    int slotEnd = slotStart + consultationTime;
                    
                    // Убеждаемся, что слот не выходит за пределы рабочего дня
                    if (slotEnd <= endMinutes)
                    {
                        string formatted = $"{slotStart / 60:D2}:{slotStart % 60:D2}-{slotEnd / 60:D2}:{slotEnd % 60:D2}";
                        result.Add(formatted);
                    }
                }
            }
            
            return result.ToArray();
        }
    }
}















using System;
using Microsoft.VisualStudio.TestTools.UnitTesting;
using SF2022UserNN.Lib;

namespace SF2022UserNN.Lib.Tests
{
    [TestClass]
    public class CalculationsTests
    {
        private Calculations _calc;
        
        [TestInitialize]
        public void Setup()
        {
            _calc = new Calculations();
        }
        
        // ТЕСТ 1: Базовый пример из задания
        [TestMethod]
        public void Test1_BasicExample()
        {
            // Arrange
            var startTimes = new TimeSpan[]
            {
                TimeSpan.FromHours(10),  // 10:00
                TimeSpan.FromHours(11),  // 11:00
                TimeSpan.FromHours(15),  // 15:00
                TimeSpan.FromHours(15).Add(TimeSpan.FromMinutes(30)), // 15:30
                TimeSpan.FromHours(16).Add(TimeSpan.FromMinutes(50))  // 16:50
            };
            var durations = new int[] { 60, 30, 10, 10, 40 };
            var begin = TimeSpan.FromHours(8);
            var end = TimeSpan.FromHours(18);
            int consultation = 30;
            
            // Act
            var result = _calc.AvailablePeriods(startTimes, durations, begin, end, consultation);
            
            // Assert
            var expected = new string[]
            {
                "08:00-08:30", "08:30-09:00", "09:00-09:30", "09:30-10:00",
                "11:30-12:00", "12:00-12:30", "12:30-13:00", "13:00-13:30",
                "13:30-14:00", "14:00-14:30", "14:30-15:00", "15:40-16:10"
            };
            
            CollectionAssert.AreEqual(expected, result);
        }
        
        // ТЕСТ 2: Недостаточно времени перед первой консультацией
        [TestMethod]
        public void Test2_InsufficientTimeBeforeFirst()
        {
            var startTimes = new TimeSpan[] { TimeSpan.FromHours(9).Add(TimeSpan.FromMinutes(30)) };
            var durations = new int[] { 30 };
            var begin = TimeSpan.FromHours(9);
            var end = TimeSpan.FromHours(18);
            int consultation = 45;
            
            var result = _calc.AvailablePeriods(startTimes, durations, begin, end, consultation);
            
            // Должно быть пусто, так как до 9:30 всего 30 минут, а нужно 45
            Assert.AreEqual(0, result.Length);
        }
        
        // ТЕСТ 3: Недостаточно времени после последней консультации
        [TestMethod]
        public void Test3_InsufficientTimeAfterLast()
        {
            var startTimes = new TimeSpan[] { TimeSpan.FromHours(16).Add(TimeSpan.FromMinutes(40)) };
            var durations = new int[] { 30 };
            var begin = TimeSpan.FromHours(9);
            var end = TimeSpan.FromHours(17).Add(TimeSpan.FromMinutes(30));
            int consultation = 30;
            
            var result = _calc.AvailablePeriods(startTimes, durations, begin, end, consultation);
            
            // Консультация с 16:40 до 17:10, до конца дня (17:30) остается 20 минут - недостаточно
            Assert.AreEqual(0, result.Length);
        }
        
        // ТЕСТ 4: Недостаточно времени между консультациями
        [TestMethod]
        public void Test4_InsufficientTimeBetween()
        {
            var startTimes = new TimeSpan[]
            {
                TimeSpan.FromHours(10),
                TimeSpan.FromHours(10).Add(TimeSpan.FromMinutes(50))
            };
            var durations = new int[] { 30, 30 };
            var begin = TimeSpan.FromHours(9);
            var end = TimeSpan.FromHours(18);
            int consultation = 30;
            
            var result = _calc.AvailablePeriods(startTimes, durations, begin, end, consultation);
            
            // Проверяем, что нет слота 10:30-11:00 (промежуток всего 20 минут)
            Assert.IsFalse(result.Contains("10:30-11:00"));
        }
        
        // ТЕСТ 5: Разная длительность консультаций
        [TestMethod]
        public void Test5_DifferentConsultationTimes()
        {
            var startTimes = new TimeSpan[] { TimeSpan.FromHours(12) };
            var durations = new int[] { 60 };
            var begin = TimeSpan.FromHours(9);
            var end = TimeSpan.FromHours(18);
            
            // 15-минутные слоты
            var result15 = _calc.AvailablePeriods(startTimes, durations, begin, end, 15);
            // 60-минутные слоты
            var result60 = _calc.AvailablePeriods(startTimes, durations, begin, end, 60);
            
            // 15-минутных слотов должно быть много
            Assert.IsTrue(result15.Length > 0);
            
            // 60-минутные слоты должны включать 09:00-10:00
            Assert.IsTrue(result60.Contains("09:00-10:00"));
            
            // 11:00-12:00 пересекается с занятым временем (12:00-13:00 занято)
            Assert.IsFalse(result60.Contains("11:00-12:00"));
        }
        
        // ТЕСТ 6: Пустое расписание
        [TestMethod]
        public void Test6_EmptySchedule()
        {
            var result = _calc.AvailablePeriods(
                new TimeSpan[0], new int[0],
                TimeSpan.FromHours(9), TimeSpan.FromHours(17), 60);
            
            var expected = new string[]
            {
                "09:00-10:00", "10:00-11:00", "11:00-12:00",
                "12:00-13:00", "13:00-14:00", "14:00-15:00",
                "15:00-16:00", "16:00-17:00"
            };
            
            CollectionAssert.AreEqual(expected, result);
        }
        
        // ТЕСТ 7: Пересекающиеся консультации
        [TestMethod]
        public void Test7_OverlappingAppointments()
        {
            var startTimes = new TimeSpan[]
            {
                TimeSpan.FromHours(10),
                TimeSpan.FromHours(10).Add(TimeSpan.FromMinutes(30)),
                TimeSpan.FromHours(11)
            };
            var durations = new int[] { 60, 30, 60 };
            var begin = TimeSpan.FromHours(9);
            var end = TimeSpan.FromHours(17);
            int consultation = 30;
            
            var result = _calc.AvailablePeriods(startTimes, durations, begin, end, consultation);
            
            // Проверяем наличие слотов
            Assert.IsTrue(result.Contains("09:00-09:30"));
            Assert.IsTrue(result.Contains("09:30-10:00"));
            Assert.IsFalse(result.Contains("10:00-10:30")); // Занято
            Assert.IsFalse(result.Contains("11:30-12:00")); // Занято
            Assert.IsTrue(result.Contains("12:30-13:00"));  // Свободно
        }
        
        // ТЕСТ 8: Время консультации 0 или отрицательное
        [TestMethod]
        public void Test8_ZeroOrNegativeConsultation()
        {
            var startTimes = new TimeSpan[] { TimeSpan.FromHours(10) };
            var durations = new int[] { 30 };
            var begin = TimeSpan.FromHours(9);
            var end = TimeSpan.FromHours(17);
            
            var resultZero = _calc.AvailablePeriods(startTimes, durations, begin, end, 0);
            var resultNegative = _calc.AvailablePeriods(startTimes, durations, begin, end, -15);
            
            Assert.AreEqual(0, resultZero.Length);
            Assert.AreEqual(0, resultNegative.Length);
        }
        
        // ТЕСТ 9: Точное соответствие интервалов
        [TestMethod]
        public void Test9_ExactFitIntervals()
        {
            var startTimes = new TimeSpan[]
            {
                TimeSpan.FromHours(10),
                TimeSpan.FromHours(11)
            };
            var durations = new int[] { 30, 30 };
            var begin = TimeSpan.FromHours(9);
            var end = TimeSpan.FromHours(12);
            int consultation = 30;
            
            var result = _calc.AvailablePeriods(startTimes, durations, begin, end, consultation);
            
            var expected = new string[]
            {
                "09:00-09:30", "09:30-10:00",
                "10:30-11:00", "11:30-12:00"
            };
            
            CollectionAssert.AreEqual(expected, result);
        }
        
        // ТЕСТ 10: Консультация длиннее рабочего дня
        [TestMethod]
        public void Test10_ConsultationLongerThanWorkday()
        {
            var result = _calc.AvailablePeriods(
                new TimeSpan[0], new int[0],
                TimeSpan.FromHours(9), TimeSpan.FromHours(10), 90);
            
            Assert.AreEqual(0, result.Length);
        }
        
        // ТЕСТ 11: Дополнительный тест - консультации вплотную (без промежутков)
        [TestMethod]
        public void Test11_BackToBackAppointments()
        {
            var startTimes = new TimeSpan[]
            {
                TimeSpan.FromHours(10),
                TimeSpan.FromHours(11)
            };
            var durations = new int[] { 60, 60 }; // 10:00-11:00 и 11:00-12:00
            var begin = TimeSpan.FromHours(9);
            var end = TimeSpan.FromHours(13);
            int consultation = 30;
            
            var result = _calc.AvailablePeriods(startTimes, durations, begin, end, consultation);
            
            // Свободно: 09:00-10:00 (два 30-минутных слота) и 12:00-13:00 (два слота)
            var expected = new string[]
            {
                "09:00-09:30", "09:30-10:00",
                "12:00-12:30", "12:30-13:00"
            };
            
            CollectionAssert.AreEqual(expected, result);
        }
    }
}
