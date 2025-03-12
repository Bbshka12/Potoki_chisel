# Potoki_chisel
using System;
using System.IO;
using System.IO.Pipes;
using System.Threading;

class Program
{
    private static int sum = 0;
    private static readonly Mutex mutex = new Mutex();
    private const int TARGET_SUM = 10000;

    static void Main()
    {
        string pipeName = @"\\.\pipe\MyPipe";

        // Сервер слушает и создает новый NamedPipeServerStream для каждого клиента
        while (sum < TARGET_SUM)
        {
            // Теперь мы устанавливаем максимальное количество экземпляров пайпа на неограниченное количество
            NamedPipeServerStream pipeServer = new NamedPipeServerStream(pipeName, PipeDirection.InOut, NamedPipeServerStream.MaxAllowedServerInstances, PipeTransmissionMode.Message, PipeOptions.None);

            Console.WriteLine("Server started. Waiting for clients...");

            try
            {
                // Ожидаем подключения клиента
                pipeServer.WaitForConnection();
                Console.WriteLine("Client connected.");

                // Создаем новый поток для обработки клиента
                Thread clientThread = new Thread(() => HandleClient(pipeServer));
                clientThread.Start();
            }
            catch (IOException e)
            {
                Console.WriteLine("Error: " + e.Message);
            }
        }

        Console.WriteLine("Server stopped. Final sum: " + sum);
    }

    static void HandleClient(NamedPipeServerStream pipeServer)
    {
        using (pipeServer)
        {
            try
            {
                // Чтение числа от клиента
                using (BinaryReader reader = new BinaryReader(pipeServer))
                {
                    while (pipeServer.IsConnected)
                    {
                        int number = reader.ReadInt32();

                        // Защита доступа к общей переменной (сумма)
                        mutex.WaitOne();
                        sum += number;
                        Console.WriteLine($"Received {number}. Total sum: {sum}");
                        mutex.ReleaseMutex();

                        if (sum >= TARGET_SUM)
                        {
                            break;
                        }
                    }
                }
            }
            catch (IOException e)
            {
                Console.WriteLine("Client disconnected or error occurred: " + e.Message);
            }
            finally
            {
                pipeServer.Close();
            }
        }
    }
}
