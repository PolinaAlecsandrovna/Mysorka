namespace ConsoleApp1
{
    using System.Collections.Generic;
    using System.Linq;
    using System;
    internal class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Выберите задачу для запуска (1, 2, 3, 4 или 0 для выхода):");

            while (true)
            {
                Console.Write("Введите номер задачи: ");
                string choice = Console.ReadLine();

                switch (choice)
                {
                    case "1":
                        Console.WriteLine("--- Задача 1: Цепочка чисел ---");

                        Task1.Solve();
                        break;
                    case "2":
                        Console.WriteLine("--- Задача 2: Обратная польская запись ---");
                        Task2.Solve();
                        break;
                    case "3":
                        Console.WriteLine("--- Задача 3: Анаграммы ---");
                        Task3.Solve();
                        break;
                    case "4":
                        Console.WriteLine("--- Задача 4: Лабиринт ---");
                        Task4.Solve();
                        break;
                    case "0":
                        return;
                    default:
                        Console.WriteLine("Неверный ввод.");
                        break;
                }
                Console.WriteLine();
            }
        }
    }



    public class Task1
    {
        public static void Solve()
        {
            string input = Console.ReadLine();
            if (string.IsNullOrEmpty(input)) return;

            string[] numbers = input.Split(new[] { ' ' }, StringSplitOptions.RemoveEmptyEntries);

            var groups = new Dictionary<int, List<string>>();

            foreach (var numStr in numbers) {
                int prod = (numStr[0] - '0') * (numStr[1] - '0');

                if (!groups.ContainsKey(prod)) groups[prod] = new List<string>();
                groups[prod].Add(numStr);

            }
            var bestGroup = groups.Values.OrderByDescending(g => g.Count).First();
            Console.WriteLine($"{bestGroup.Count}{string.Join(" ", bestGroup)}");
        }

    }
    public class Task2
    {
        public static void Solve()
        {
            string

        }

    }
    public class Task3
    {
        public static void Solve()
        {
            string s1 = Console.ReadLine();
            string s2 = Console.ReadLine();

            if (s1 == null || s2 == null) return;

            var sorted1 = new string(s1.OrderBy(c => c).ToArray());
            var sorted2 = new string(s2.OrderBy(c => c).ToArray());

            if (sorted1 == sorted2)
                Console.WriteLine("Да");
            else
                Console.WriteLine("Нет");
        }
    }
    public class Task4
    {
        public static void Solve()
        {

        }

    }
}


