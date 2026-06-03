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
            int endMinutes = (int)endWorkingTime.TotalMinutes;  // ← endMinutes здесь
            
            // Convert appointments to (start, end) in minutes
            var appointments = new List<(int Start, int End)>();
            for (int i = 0; i < startTimes.Length; i++)
            {
                int startMinutes = (int)startTimes[i].TotalMinutes;
                int appointmentEndMinutes = startMinutes + durations[i];  // ← переименовано
                appointments.Add((startMinutes, appointmentEndMinutes));
            }
            
            // Sort appointments by start time
            appointments = appointments.OrderBy(a => a.Start).ToList();
            
            // Merge overlapping or adjacent appointments
            var merged = new List<(int Start, int End)>();
            foreach (var appointment in appointments)
            {
                if (merged.Count == 0 || appointment.Start > merged[merged.Count - 1].End)
                {
                    merged.Add(appointment);
                }
                else
                {
                    var last = merged[merged.Count - 1];
                    merged[merged.Count - 1] = (last.Start, Math.Max(last.End, appointment.End));
                }
            }
            
            // Find free slots
            var freeSlots = new List<(int Start, int End)>();
            int currentTime = beginMinutes;
            
            // Check before first appointment
            if (merged.Count > 0 && merged[0].Start > currentTime)
            {
                if (merged[0].Start - currentTime >= consultationTime)
                {
                    freeSlots.Add((currentTime, merged[0].Start));
                }
            }
            
            // Check between appointments
            for (int i = 0; i < merged.Count - 1; i++)
            {
                int slotStart = merged[i].End;
                int slotEnd = merged[i + 1].Start;
                if (slotEnd - slotStart >= consultationTime)
                {
                    freeSlots.Add((slotStart, slotEnd));
                }
            }
            
            // Check after last appointment
            if (merged.Count > 0)
            {
                currentTime = merged[merged.Count - 1].End;
            }
            if (endMinutes - currentTime >= consultationTime)
            {
                freeSlots.Add((currentTime, endMinutes));
            }
            
            // If no appointments, check whole day
            if (merged.Count == 0)
            {
                if (endMinutes - beginMinutes >= consultationTime)
                {
                    freeSlots.Add((beginMinutes, endMinutes));
                }
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
