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

            foreach (var numStr in numbers)
            {
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
            string input = Console.ReadLine();
            if (string.IsNullOrWhiteSpace(input)) return;

            string[] tokens = input.Split(new[] { ' ' }, StringSplitOptions.RemoveEmptyEntries);
            Stack<double> stack = new Stack<double>();

            try
            {
                foreach (var token in tokens)
                {
                    if (token == "+" || token == "-" || token == "*" || token == "/" || token == "^")
                    {
                        if (stack.Count < 2) throw new Exception("Недостаточно операндов");

                        double b = stack.Pop();
                        double a = stack.Pop();

                        switch (token)
                        {
                            case "+": stack.Push(a + b); break;
                            case "-": stack.Push(a - b); break;
                            case "*": stack.Push(a * b); break;
                            case "/":
                                if (b == 0) throw new DivideByZeroException();
                                stack.Push(a / b);
                                break;
                            case "^": stack.Push(Math.Pow(a, b)); break;
                        }
                    }
                    else
                    {
                        if (double.TryParse(token, NumberStyles.Any, CultureInfo.InvariantCulture, out double num))
                        {
                            stack.Push(num);
                        }
                        else
                        {
                            throw new FormatException("Неверный токен");
                        }
                    }
                }

                if (stack.Count == 1)
                {
                    double result = stack.Pop();
                    if (result % 1 == 0)
                        Console.WriteLine((long)result);
                    else
                        Console.WriteLine(result.ToString(CultureInfo.InvariantCulture));
                }
                else
                {
                    Console.WriteLine("NULL");
                }
            }
            catch
            {
                Console.WriteLine("NULL");
            }
        }

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
        
        
        public class Point
    {
        public int X, Y;
        public Point(int x, int y) { X = x; Y = y; }

        public override bool Equals(object obj) => obj is Point p && X == p.X && Y == p.Y;
        public override int GetHashCode() => HashCode.Combine(X, Y);
    }

    public static void Solve()
    {
        var dims = Console.ReadLine().Split().Select(int.Parse).ToArray();
        int M = dims[0], N = dims[1];

        int[,] maze = new int[M, N];
        for (int i = 0; i < M; i++)
        {
            var row = Console.ReadLine().Split().Select(int.Parse).ToArray();
            for (int j = 0; j < N; j++) maze[i, j] = row[j];
        }

        var startCoords = Console.ReadLine().Split().Select(int.Parse).ToArray();
        Point start = new Point(startCoords[0], startCoords[1]);

        var endCoords = Console.ReadLine().Split().Select(int.Parse).ToArray();
        Point end = new Point(endCoords[0], endCoords[1]);

        if (maze[start.X, start.Y] == 1 || maze[end.X, end.Y] == 1)
        {
            Console.WriteLine("None");
            return;
        }

        Queue<Point> queue = new Queue<Point>();
        queue.Enqueue(start);

        var cameFrom = new Dictionary<Point, Point>();
        cameFrom[start] = null; 

        bool found = false;

        int[] dx = { -1, 1, 0, 0 };
        int[] dy = { 0, 0, -1, 1 };

        while (queue.Count > 0)
        {
            Point current = queue.Dequeue();

            if (current.X == end.X && current.Y == end.Y)
            {
                found = true;
                break;
            }

            for (int i = 0; i < 4; i++)
            {
                int nx = current.X + dx[i];
                int ny = current.Y + dy[i];

                if (nx >= 0 && nx < M && ny >= 0 && ny < N && maze[nx, ny] == 0)
                {
                    Point neighbor = new Point(nx, ny);
                    if (!cameFrom.ContainsKey(neighbor))
                    {
                        queue.Enqueue(neighbor);
                        cameFrom[neighbor] = current; 
                    }
                }
            }
        }

        if (found)
        {
            List<Point> path = new List<Point>();
            Point curr = end;
            while (curr != null)
            {
                path.Add(curr);
                curr = cameFrom[curr];
            }
            path.Reverse(); 

            foreach (var p in path)
            {
                Console.WriteLine($"({p.X},{p.Y})");
            }
        }
        else
        {
            Console.WriteLine("None");
        }
    }

}

