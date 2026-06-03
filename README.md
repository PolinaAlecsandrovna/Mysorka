using System;
using System.Collections.Generic;
using System.Linq;

namespace SF2022UserNN.Lib
{
    public class Calculations
    {
        /// <summary>
        /// Returns available time slots of consultationTime minutes.
        /// </summary>
        /// <param name="startTimes">Array of appointment start times</param>
        /// <param name="durations">Array of appointment durations in minutes</param>
        /// <param name="beginWorkingTime">Start of working day</param>
        /// <param name="endWorkingTime">End of working day</param>
        /// <param name="consultationTime">Required consultation duration in minutes</param>
        /// <returns>Array of strings in format "HH:mm-HH:mm"</returns>
        public string[] AvailablePeriods(
            TimeSpan[] startTimes,
            int[] durations,
            TimeSpan beginWorkingTime,
            TimeSpan endWorkingTime,
            int consultationTime)
        {
            // Validate input
            if (consultationTime <= 0)
                return Array.Empty<string>();
            
            if (startTimes == null || durations == null || 
                startTimes.Length != durations.Length)
                return Array.Empty<string>();
            
            // Convert working day to minutes
            int beginMinutes = (int)beginWorkingTime.TotalMinutes;
            int endMinutes = (int)endWorkingTime.TotalMinutes;
            
            // Convert appointments to (start, end) in minutes
            var appointments = new List<(int Start, int End)>();
            for (int i = 0; i < startTimes.Length; i++)
            {
                int startMinutes = (int)startTimes[i].TotalMinutes;
                int appointmentEndMinutes = startMinutes + durations[i];
                
                // Only add appointments that are within working hours
                if (appointmentEndMinutes > beginMinutes && startMinutes < endMinutes)
                {
                    appointments.Add((Math.Max(startMinutes, beginMinutes), 
                                     Math.Min(appointmentEndMinutes, endMinutes)));
                }
            }
            
            // Sort appointments by start time
            appointments = appointments.OrderBy(a => a.Start).ToList();
            
            // Merge ONLY overlapping appointments (not adjacent)
            var merged = new List<(int Start, int End)>();
            foreach (var appointment in appointments)
            {
                if (merged.Count == 0 || appointment.Start > merged[merged.Count - 1].End)
                {
                    merged.Add(appointment);
                }
                else if (appointment.Start <= merged[merged.Count - 1].End)
                {
                    var last = merged[merged.Count - 1];
                    merged[merged.Count - 1] = (last.Start, Math.Max(last.End, appointment.End));
                }
            }
            
            // Find free slots
            var freeSlots = new List<(int Start, int End)>();
            int currentTime = beginMinutes;
            
            // Add all free intervals
            foreach (var appointment in merged)
            {
                if (currentTime < appointment.Start)
                {
                    freeSlots.Add((currentTime, appointment.Start));
                }
                currentTime = Math.Max(currentTime, appointment.End);
            }
            
            // Add remaining time after last appointment
            if (currentTime < endMinutes)
            {
                freeSlots.Add((currentTime, endMinutes));
            }
            
            // Split free slots into consultationTime intervals
            var result = new List<string>();
            foreach (var slot in freeSlots)
            {
                int current = slot.Start;
                while (current + consultationTime <= slot.End)
                {
                    int startHour = current / 60;
                    int startMin = current % 60;
                    int endHour = (current + consultationTime) / 60;
                    int endMin = (current + consultationTime) % 60;
                    
                    result.Add($"{startHour:D2}:{startMin:D2}-{endHour:D2}:{endMin:D2}");
                    current += consultationTime;
                }
            }
            
            return result.ToArray();
        }
    }
}














using System;
using Microsoft.VisualStudio.TestTools.UnitTesting;
using SF2022UserNN.Lib;
using System.Linq;

namespace SF2022UserNN.Lib.Tests
{
    [TestClass]
    public class CalculationsTests
    {
        private Calculations _calculations;
        
        [TestInitialize]
        public void Setup()
        {
            _calculations = new Calculations();
        }
        
        [TestMethod]
        public void AvailablePeriods_BasicCaseFromExample_ReturnsCorrectSlots()
        {
            // Arrange
            TimeSpan[] startTimes = new[]
            {
                TimeSpan.FromHours(10),
                TimeSpan.FromHours(11),
                TimeSpan.FromHours(15),
                TimeSpan.FromHours(15).Add(TimeSpan.FromMinutes(30)),
                TimeSpan.FromHours(16).Add(TimeSpan.FromMinutes(50))
            };
            int[] durations = { 60, 30, 10, 10, 40 };
            TimeSpan beginWorkingTime = TimeSpan.FromHours(8);
            TimeSpan endWorkingTime = TimeSpan.FromHours(18);
            int consultationTime = 30;
            
            // Act
            string[] result = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, consultationTime);
            
            // Assert
            string[] expected = new[]
            {
                "08:00-08:30", "08:30-09:00", "09:00-09:30", "09:30-10:00",
                "11:30-12:00", "12:00-12:30", "12:30-13:00", "13:00-13:30",
                "13:30-14:00", "14:00-14:30", "14:30-15:00", "15:40-16:10"
            };
            
            CollectionAssert.AreEqual(expected, result);
        }
        
        [TestMethod]
        public void AvailablePeriods_InsufficientTimeBeforeFirstAppointment_ReturnsEmpty()
        {
            // Arrange - только 30 минут до первой консультации, нужно 45
            TimeSpan[] startTimes = new[] { TimeSpan.FromHours(9).Add(TimeSpan.FromMinutes(30)) };
            int[] durations = new[] { 30 };
            TimeSpan beginWorkingTime = TimeSpan.FromHours(9);
            TimeSpan endWorkingTime = TimeSpan.FromHours(18);
            int consultationTime = 45;
            
            // Act
            string[] result = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, consultationTime);
            
            // Assert
            Assert.AreEqual(0, result.Length);
        }
        
        [TestMethod]
        public void AvailablePeriods_InsufficientTimeAfterLastAppointment_ReturnsEmpty()
        {
            // Arrange - после последней консультации 20 минут, нужно 30
            TimeSpan[] startTimes = new[] { TimeSpan.FromHours(16).Add(TimeSpan.FromMinutes(40)) };
            int[] durations = new[] { 30 };
            TimeSpan beginWorkingTime = TimeSpan.FromHours(9);
            TimeSpan endWorkingTime = TimeSpan.FromHours(17).Add(TimeSpan.FromMinutes(30));
            int consultationTime = 30;
            
            // Act
            string[] result = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, consultationTime);
            
            // Assert - после 17:10 нет времени для 30-минутной консультации
            // Последняя консультация заканчивается в 17:10, рабочий день до 17:30
            // Остается 20 минут, недостаточно
            Assert.IsFalse(result.Contains("17:10-17:40"));
        }
        
        [TestMethod]
        public void AvailablePeriods_InsufficientTimeBetweenAppointments_NoSlotsInGap()
        {
            // Arrange - промежуток между консультациями 20 минут, нужно 30
            TimeSpan[] startTimes = new[]
            {
                TimeSpan.FromHours(10),
                TimeSpan.FromHours(10).Add(TimeSpan.FromMinutes(50))
            };
            int[] durations = new[] { 30, 30 };
            TimeSpan beginWorkingTime = TimeSpan.FromHours(9);
            TimeSpan endWorkingTime = TimeSpan.FromHours(18);
            int consultationTime = 30;
            
            // Act
            string[] result = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, consultationTime);
            
            // Assert - промежуток 10:30-10:50 = 20 минут, недостаточно
            Assert.IsFalse(result.Contains("10:30-11:00"));
        }
        
        [TestMethod]
        public void AvailablePeriods_DifferentConsultationDurations_WorksCorrectly()
        {
            // Arrange
            TimeSpan[] startTimes = new[] { TimeSpan.FromHours(12) };
            int[] durations = new[] { 60 };
            TimeSpan beginWorkingTime = TimeSpan.FromHours(9);
            TimeSpan endWorkingTime = TimeSpan.FromHours(18);
            
            // Act - консультация 15 минут
            string[] result15 = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, 15);
            
            // Act - консультация 60 минут
            string[] result60 = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, 60);
            
            // Assert
            Assert.IsTrue(result15.Length > 0);
            Assert.IsTrue(result60.Contains("09:00-10:00"));
            Assert.IsFalse(result60.Contains("11:00-12:00")); // Пересекается с занятым временем
        }
        
        [TestMethod]
        public void AvailablePeriods_EmptySchedule_ReturnsFullDaySlots()
        {
            // Arrange
            TimeSpan[] startTimes = Array.Empty<TimeSpan>();
            int[] durations = Array.Empty<int>();
            TimeSpan beginWorkingTime = TimeSpan.FromHours(9);
            TimeSpan endWorkingTime = TimeSpan.FromHours(17);
            int consultationTime = 60;
            
            // Act
            string[] result = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, consultationTime);
            
            // Assert
            string[] expected = new[]
            {
                "09:00-10:00", "10:00-11:00", "11:00-12:00",
                "12:00-13:00", "13:00-14:00", "14:00-15:00",
                "15:00-16:00", "16:00-17:00"
            };
            
            CollectionAssert.AreEqual(expected, result);
        }
        
        [TestMethod]
        public void AvailablePeriods_OverlappingAppointments_HandledAsSingleBlock()
        {
            // Arrange - консультации с 10:00 до 12:30
            TimeSpan[] startTimes = new[]
            {
                TimeSpan.FromHours(10),
                TimeSpan.FromHours(10).Add(TimeSpan.FromMinutes(30)),
                TimeSpan.FromHours(11)
            };
            int[] durations = new[] { 60, 30, 60 };
            TimeSpan beginWorkingTime = TimeSpan.FromHours(9);
            TimeSpan endWorkingTime = TimeSpan.FromHours(17);
            int consultationTime = 30;
            
            // Act
            string[] result = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, consultationTime);
            
            // Assert
            Assert.IsTrue(result.Contains("09:00-09:30"));
            Assert.IsTrue(result.Contains("09:30-10:00"));
            Assert.IsFalse(result.Contains("10:00-10:30")); // Занято
            Assert.IsFalse(result.Contains("11:30-12:00")); // Занято
            Assert.IsTrue(result.Contains("12:30-13:00")); // Свободно
        }
        
        [TestMethod]
        public void AvailablePeriods_ConsultationTimeZeroOrNegative_ReturnsEmpty()
        {
            // Arrange
            TimeSpan[] startTimes = new[] { TimeSpan.FromHours(10) };
            int[] durations = new[] { 30 };
            TimeSpan beginWorkingTime = TimeSpan.FromHours(9);
            TimeSpan endWorkingTime = TimeSpan.FromHours(17);
            
            // Act - консультация 0 минут
            string[] resultZero = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, 0);
            
            // Act - консультация отрицательная
            string[] resultNegative = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, -15);
            
            // Assert
            Assert.AreEqual(0, resultZero.Length);
            Assert.AreEqual(0, resultNegative.Length);
        }
        
        [TestMethod]
        public void AvailablePeriods_ExactFitIntervals_ReturnsCorrectSlots()
        {
            // Arrange
            TimeSpan[] startTimes = new[]
            {
                TimeSpan.FromHours(10),
                TimeSpan.FromHours(11)
            };
            int[] durations = new[] { 30, 30 };
            TimeSpan beginWorkingTime = TimeSpan.FromHours(9);
            TimeSpan endWorkingTime = TimeSpan.FromHours(12);
            int consultationTime = 30;
            
            // Act
            string[] result = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, consultationTime);
            
            // Assert
            string[] expected = new[]
            {
                "09:00-09:30", "09:30-10:00",
                "10:30-11:00", "11:30-12:00"
            };
            
            CollectionAssert.AreEqual(expected, result);
        }
        
        [TestMethod]
        public void AvailablePeriods_ConsultationLongerThanWorkingDay_ReturnsEmpty()
        {
            // Arrange
            TimeSpan[] startTimes = Array.Empty<TimeSpan>();
            int[] durations = Array.Empty<int>();
            TimeSpan beginWorkingTime = TimeSpan.FromHours(9);
            TimeSpan endWorkingTime = TimeSpan.FromHours(10); // Рабочий день 1 час
            int consultationTime = 90; // Нужно 1.5 часа
            
            // Act
            string[] result = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, consultationTime);
            
            // Assert
            Assert.AreEqual(0, result.Length);
        }
        
        [TestMethod]
        public void AvailablePeriods_AppointmentsAtBoundaries_HandlesCorrectly()
        {
            // Arrange - консультации в начале и в конце дня
            TimeSpan[] startTimes = new[]
            {
                TimeSpan.FromHours(9),
                TimeSpan.FromHours(17)
            };
            int[] durations = new[] { 30, 30 };
            TimeSpan beginWorkingTime = TimeSpan.FromHours(9);
            TimeSpan endWorkingTime = TimeSpan.FromHours(17).Add(TimeSpan.FromMinutes(30));
            int consultationTime = 30;
            
            // Act
            string[] result = _calculations.AvailablePeriods(
                startTimes, durations, beginWorkingTime, endWorkingTime, consultationTime);
            
            // Assert
            Assert.IsFalse(result.Contains("09:00-09:30")); // Занято
            Assert.IsTrue(result.Contains("09:30-10:00"));
            Assert.IsTrue(result.Contains("16:30-17:00"));
            Assert.IsFalse(result.Contains("17:00-17:30")); // Занято
            Assert.IsTrue(result.Contains("17:30-18:00"));
        }
    }
}



