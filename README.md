Лабораторна робота №3
Тема: Lab 03 - Адреси, а не імена: вказівники й пам'ять

Студент: Свириденко Павло Володимирович
Група: Б-F2-26-2-ПІ

1. Інструкції ISA
Опкод	Мнемоніка	Розмір	Прапорці	Що робить
0x20	LOADI A, imm8	2	-	A = imm8
0x21	LOADI B, imm8	2	-	B = imm8
0x22	LOAD A, [addr16]	3	-	A = mem[addr]
0x23	LOAD B, [addr16]	3	-	B = mem[addr]
0x24	STORE [addr16], A	3	-	mem[addr] = A
0x25	STORE [addr16], B	3	-	mem[addr] = B
0x26	MOV A, B	1	-	A = B
0x27	MOV B, A	1	-	B = A
0x28	LOADH H, imm16	3	-	H = imm16
0x29	LOAD A, [H]	1	-	A = mem[H]
0x2A	STORE [H], A	1	-	mem[H] = A
0x2B	INCH	1	-	H = H + 1
0x2C	DECH	1	-	H = H - 1

Додатково:
0x02	OUT	1	-	надрукувати A як символ
0x03	OUTN	1	-	надрукувати A як десяткове число і пробіл

Ще додав регістр H (16 біт), set16 (як get16, тільки навпаки), команду run (крокує до HALT) і команди get16, set16, load, reset для оболонки.

2. Порядок байтів
ember> set16 0x10 0x1234 - пишемо 0x1234 за адресою 0x10
ember> dump - дивимось пам'ять
0010  34 12 00 00 00 00 00 00 00 00 00 00 00 00 00 00  |4...............|

Першим лежить молодший байт 0x34, потім старший 0x12. В ASCII справа видно 4, бо 0x34 це код цифри 4.

3. Програма 1 (дані за адресою)
ember> load 0x800 65 66 67 0 - кладу дані ABC на 0x800
ember> load 0 0x22 0x00 0x08 0x02 0x00 - кладу код з адреси 0
ember> run - виконується до HALT
A

0x22 0x00 0x08 це LOAD A, [0x0800] (адреса теж молодшим байтом уперед), 0x02 це OUT, 0x00 це HALT. Код з 0x000, дані з 0x800.

4. Програма 2 (те саме через H)
ember> reset - скидаю PC, H і halted
ember> load 0 0x28 0x00 0x08 0x29 0x02 0x2B 0x29 0x02 0x00 - LOADH, LOAD A,[H], OUT, INCH, LOAD A,[H], OUT, HALT
ember> step
ember> step
ember> regs
PC:4 H:2048 A:65 B:0 Z:0 N:0 C:0 - H=2048 (це 0x0800), в A літера A (65)
ember> step
Aember> step - третій step це OUT, він надрукував A (тому в одному рядку з ember>)
ember> regs
PC:6 H:2049 A:65 B:0 Z:0 N:0 C:0 - четвертий step це INCH, H став на 1 більше
ember> run
B - далі LOAD A,[H] дав 66 і OUT надрукував B

Вийшло AB. Різниця з першою програмою в тому, що адреса тепер в регістрі H і її можна рухати через INCH, а не прописувати в самій інструкції.

5. Звіт AddressSanitizer
Тимчасово дописав у main два рядки:
volatile std::size_t i = MEM_SIZE;
mem.data[i] = 1;

Зібрав у Debug (cmake -DCMAKE_BUILD_TYPE=Debug ..) і запустив:

src/main.cpp:44:15: runtime error: index 4096 out of bounds for type 'unsigned char [4096]'
==25328==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7658f3201660
WRITE of size 1 at 0x7658f3201660 thread T0
#0 0x60f29bbfb173 in main src/main.cpp:44
[1632, 5728) 'mem' (line 42) <== Memory access at offset 5728 overflows this variable
SUMMARY: AddressSanitizer: stack-buffer-overflow src/main.cpp:44 in main

В масиві data 4096 байтів, тобто індекси 0..4095, а я писав в індекс 4096 (на один далі). ASan показав рядок 44 і змінну mem. Ці два рядки я потім прибрав. Перевірка меж лишилась: mem_get і mem_set не лізуть за масив, а step зупиняє машину з повідомленням, якщо адреса за межами пам'яті.

6. Вказівник хоста і адреса гостя
У CPU є вказівник на Memory (у коді mem). Це вказівник хоста, тобто мого C++ процесу, він показує де лежить вся пам'ять. А pc і h це просто 16-бітні числа, адреси гостя, тобто всередині ember. Нормальні адреси 0..4095, інші step не пускає. Їх видно в regs, і програма може з ними рахувати (INCH додає до H одиницю).

Вказівник хоста на Byte для гостя нічого не значить і щоразу інший. Якщо його покласти в пам'ять гостя, то на іншій машині або після перевиділення пам'яті він буде вказувати невідомо куди. Тому pc і h зроблені індексами, а не вказівниками.