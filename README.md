9using System;
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

3 Назначение разработки
3.1 Функциональное назначение
Программный продукт задуман с целью обеспечения пользователей единым информационным пространством для ведения личных задач, отслеживания занятости друзей и совместного планирования общих дел. Приложение должно автоматизировать процесс согласования времени и обеспечить своевременное информирование участников о предстоящих событиях.

3.2 Эксплуатационное назначение
Мобильное приложение «GoPlan» должно удовлетворять следующие важные эксплуатационные назначения: ведение личного ежедневника, управление списком друзей и заявками, просмотр задач друзей (с учетом настроек приватности), создание и управление совместными задачами, настройка push-уведомлений и напоминаний.

4 Требования к программе или программному изделию
4.1 Требования к функциональным характеристикам

4.1.1 Требования к составу выполняемых функций
Программный продукт должен обеспечивать возможность выполнения перечисленных ниже функций:

регистрация и авторизация пользователей с разграничением ролей (пользователь, техподдержка, администратор);

создание, редактирование, удаление и просмотр личных задач (CRUD);

настройка напоминаний для личных задач (push-уведомления);

поиск пользователей по логину и отправка заявок в друзья;

просмотр списка друзей и активных задач друзей (при наличии разрешения);

создание совместных задач с выбором участников из списка друзей;

принятие или отклонение приглашений на участие в совместных задачах;

настройка профиля (изменение данных, уровня приватности, темы оформления, уведомлений);

обработка обращений пользователей через панель технической поддержки;

управление учетными записями пользователей через панель администратора (блокировка, разблокировка, удаление).

4.1.2 Требования к организации входных и выходных данных
Входные данные (логин, пароль, email, название и описание задачи, дата и время) должны быть корректны. Система должна валидировать вводимые данные: проверять обязательные поля, корректность формата email и минимальную длину пароля.
Выходные данные (списки задач, профили, уведомления) должны отображаться в удобном для пользователя формате. Передача конфиденциальных данных должна быть ограничена в соответствии с настройками приватности профиля пользователя.4.1.3 Требования к временным характеристикам
Время отклика сервера при выполнении запросов к базе данных должно быть минимальным. Загрузка экранов приложения (главная страница, списки задач) не должна превышать 2-3 секунд при стабильном интернет-соединении.

4.2 Требования к надежности

4.2.1 Требования к обеспечению надежного (устойчивого) функционирования программы
Надёжность программного средства должна соответствовать следующим требованиям:

использование лицензионного программного обеспечения и библиотек;

защита от ошибочных действий пользователей на уровне интерфейса (диалоговые окна подтверждения удаления/блокировки);

обеспечение обнаружения и диагностики ошибок с выдачей соответствующих сообщений пользователю (например, «Ошибка сети», «Неверный логин или пароль»).

4.2.2 Время восстановления после отказа
Время восстановления после отказа, вызванного сбоем электропитания устройства или потерей интернет-соединения, не должно превышать времени, необходимого на перезагрузку приложения и повторную авторизацию пользователя (при наличии сохраненной сессии — автоматически).

4.2.3 Отказы из-за некорректных действий оператора
Отказы программы возможны вследствие некорректных действий оператора при взаимодействии с интерфейсом. Во избежание возникновения отказов программы следует обеспечить работу конечного пользователя без предоставления ему административных привилегий (роль «Пользователь» не имеет доступа к панелям администратора и техподдержки).

4.3 Условия эксплуатации

4.3.1 Климатические условия эксплуатации
Климатические условия эксплуатации должны удовлетворять требованиям, предъявляемым к техническим средствам (мобильным устройствам) в части условий их эксплуатации.

4.3.2 Требования к численности и квалификации персонала
Количество персонала, требуемого для работы программы, должно составлять одну штатную единицу – конечный пользователь программы (оператор). Квалификация: уверенный пользователь мобильных устройств.

4.4 Требования к составу и параметрам технических средств
Для работы программного продукта на устройстве требуется:

мобильное устройство на базе ОС Android (версия 8.0 и выше);

объем оперативной памяти не менее 2 ГБ;

стабильное интернет-соединение.

4.5 Требования к информационной и программной совместимости
4.5.1 Требования к исходным кодам и языкам программирования
Исходные коды клиентской части программы должны быть реализованы на языке Java. Серверная часть — на языке JavaScript (Node.js). В качестве интегрированной среды разработки клиентской части должна быть использована среда Android Studio. Для серверной части — Visual Studio Code.

4.5.2 Требования к программным средствам, используемым программой
Системные программные средства, используемые программой, должны быть представлены лицензионной версией операционной системы Android. В качестве СУБД должна использоваться PostgreSQL.

4.5.3 Требования к защите информации и программ
Пароли пользователей должны храниться в базе данных в виде хешей (с использованием алгоритма bcrypt). Авторизация запросов должна осуществляться с помощью JWT-токенов.

4.6 Требования к маркировке и упаковке
Требования к маркировке и упаковке не предъявляются.

4.7 Требования к транспортировке и хранению
Программный продукт представляется в виде готового проекта (исходный код клиентской и серверной частей). В состав проекта входят:

файлы исходного кода (Java, XML, JavaScript);

скрипты создания структуры базы данных;

настройки сервера.
Конечный проект должен быть полностью подготовлен к дальнейшему развертыванию.

4.8 Специальные требования
Программное обеспечение должно иметь дружественный интерфейс, выполненный в единой цветовой гамме, защищенный от неправильных действий пользователей и снабженный системой подсказок (Toast-уведомления).

5 Требования к программной документации
Состав программной документации должен включать в себя:

техническое задание;

пояснительную записку;

руководство оператора (при необходимости).
6 Технико-экономические показатели
6.1 Предполагаемая потребность
Ориентировочная экономическая эффективность не рассчитывается.

7 Стадии и этапы разработки
7.1 Стадии разработки
Разработка должна быть проведена в три стадии:

разработка технического задания;

разработка пояснительной записки;

рабочее проектирование (написание кода, тестирование).

7.2 Этапы разработки
Этапы разработки представлены в таблице 7.1.

Таблица 7.1 — Стадии разработки

№	Название этапа	Сроки	Отчётность
1	Исследование предметной области	[даты]	Пояснительная записка
2	Выбор технологий и проектирование архитектуры	[даты]	Пояснительная записка
3	Разработка технического задания	[даты]	Техническое задание
4	Проектирование базы данных и интерфейса	[даты]	ER-диаграмма, макеты
5	Кодирование клиентской и серверной частей	[даты]	Программный продукт
6	Тестирование компонентов и сборка	[даты]	Отчет о тестировании
7	Разработка программной документации	[даты]	Пояснительная записка
8	Защита курсового проекта	[дата]	Презентация
8 Порядок контроля и приемки
8.1 Виды испытаний
Видом испытания является защита курсового проекта.

8.2 Общие требования к приемке работы
Общими требованиями к приёмке являются:

соответствие техническому заданию;

наличие пояснительной записки;

работоспособный программный продукт;

