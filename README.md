using System;
using System.Collections.Generic;
CHIP Memory20K {
    IN in[16], load, address[15];
    OUT out[16];

    PARTS:
    DMux(in=load, sel=address[14], a=loadRam, b=outThree);
    DMux(in=outThree, sel=address[13], a=loadScreen, b=loadTwo);
    DMux(in=loadTwo, sel=address[12], a=kbd, b=loadRam4k);
    RAM16K(in=in, load=loadRam, address=address[0..13], out=outRam);
    Screen(in=in, load=loadScreen, address=address[0..12], out=outScreen);
    Keyboard(out=outkbd);
    RAM4K(in=in, load=loadRam4k, address=address[0..11], out=outram4K);
    Mux16(a=outkbd, b=outram4K, sel=address[12], out=outOne);
    Mux16(a=outScreen, b=outOne, sel=address[13], out=outTwo);
    Mux16(a=outRam, b=outTwo, sel=address[14], out=out);
}

namespace ShowPicture
{
    public static class ShowPictureTask
    {
        public static string[] GenerateShowPictureCode(bool[,] pixels)
        {
            int height = pixels.GetLength(0);
            int width = pixels.GetLength(1);
            const int ramBase = 0x4000;
            const int wordsPerRow = 32;
            var lines = new List<string>();

            for (int y = 0; y < height && y < 256; y++)
            {
                for (int word = 0; word < wordsPerRow; word++)
                {
                    int x0 = word * 16;
                    lines.Add("@0");
                    lines.Add("D=A");

                    for (int bit = 0; bit < 16; bit++)
                    {
                        int x = x0 + bit;
                        if (x < width && pixels[y, x])
                        {
                            int pow = 1 << (15 - bit);
                            if (pow <= 32767)
                            {
                                lines.Add("@" + pow);
                                lines.Add("D=D+A");
                            }
                            else
                            {
                                lines.Add("@16384");
                                lines.Add("D=D+A");
                                lines.Add("@16384");
                                lines.Add("D=D+A");
                            }
                        }
                    }

                    int addr = ramBase + y * wordsPerRow + word;
                    lines.Add("@" + addr);
                    lines.Add("M=D");
                }
            }

            lines.Add("(END)");
            lines.Add("@END");
            lines.Add("0;JMP");
            return lines.ToArray();
        }
    }
}


CHIP RAM16K {
    IN in[16], load, address[14];
    OUT out[16];

    PARTS:
    DMux4Way(in=load, sel=address[12..13], a=load0, b=load1, c=load2, d=load3);
    RAM4K(in=in, load=load0, address=address[0..11], out=ram0);
    RAM4K(in=in, load=load1, address=address[0..11], out=ram1);
    RAM4K(in=in, load=load2, address=address[0..11], out=ram2);
    RAM4K(in=in, load=load3, address=address[0..11], out=ram3);
    Mux4Way16(a=ram0, b=ram1, c=ram2, d=ram3, sel=address[12..13], out=out);
}
(LOOP)
    @KBD
    D=M
    @130
    D=D-A
    @LEFT
    D;JEQ
    @UP 
    D=D-1
    D;JEQ
    @RIGHT
    D=D-1
    D;JEQ
    @DOWN
    D=D-1
    D;JEQ
    // Обнуляю текущее смещение смайлика
    @CURRENT_OFFSET
    M=0
    // Если это не UP, DOWN, LEFT, RIGHT и был нарисован смайлик, то иду его стирать. 
    @PREVIOUS_OFFSET
    D=M
    @CLEAR
    D;JNE
    // иду повторять цикл
    @LOOP
    0;JMP

(LEFT)
    // Записываю смещение в CURRENT_OFFSET
    @3978
    D=A
    @CURRENT_OFFSET
    M=D
    // иду проверять есть ли смайлик по этим координатам
    @THERE_SMILEY
    0;JMP

(UP)
    // Записываю смещение в CURRENT_OFFSET
    @2640
    D=A
    @CURRENT_OFFSET
    M=D
    // иду проверять есть ли смайлик по этим координатам
    @THERE_SMILEY
    0;JMP

(RIGHT)
    // Записываю смещение в CURRENT_OFFSET
    @3989
    D=A
    @CURRENT_OFFSET
    M=D
    // иду проверять есть ли смайлик по этим координатам
    @THERE_SMILEY
    0;JMP

(DOWN)
    // Записываю смещение в CURRENT_OFFSET
    @5296
    D=A
    @CURRENT_OFFSET
    M=D
    // иду проверять есть ли смайлик по этим координатам
    @THERE_SMILEY
    0;JMP

(THERE_SMILEY)
    // проверяю есть ли по этим координатам смайлик уже
    @CURRENT_OFFSET
    D=M
    @PREVIOUS_OFFSET
    D=D-M
    @LOOP
    D;JEQ
    // если он по другим координатам, стираю его
    @CLEAR
    0;JMP

(PAINT)
    // рисую смайлик со смещением CURRENT_OFFSET
    @CURRENT_OFFSET
    D=M
    @16384
    D=D+A
    @OFFSET
    M=D
    @7224
    D=A
    @OFFSET
    A=M
    M=D

    @CURRENT_OFFSET
    D=M
    @16416
    D=D+A
    @OFFSET
    M=D
    @7224
    D=A
    @OFFSET
    A=M
    M=D

    @CURRENT_OFFSET
    D=M
    @16448
    D=D+A
    @OFFSET
    M=D
    @7224
    D=A
    @OFFSET
    A=M
    M=D

    @CURRENT_OFFSET
    D=M
    @16544
    D=D+A
    @OFFSET
    M=D
    @24582
    D=A
    @OFFSET
    A=M
    M=D

    @CURRENT_OFFSET
    D=M
    @16576
    D=D+A
    @OFFSET
    M=D
    @14364
    D=A
    @OFFSET
    A=M
    M=D

    @CURRENT_OFFSET
    D=M
    @16608
    D=D+A
    @OFFSET
    M=D
    @4080
    D=A
    @OFFSET
    A=M
    M=D

    // запоминаю смещение в PREVIOUS_OFFSET
    @CURRENT_OFFSET
    D=M
    @PREVIOUS_OFFSET
    M=D

    // повторяю цикл
    @LOOP
    0;JMP

(CLEAR)
    // обнуляю рисунок по смещение PREVIOUS_OFFSET
    @PREVIOUS_OFFSET
    D=M
    @16384
    A=D+A
    M=0
    @16416
    A=D+A
    M=0
    @16448
    A=D+A
    M=0
    @16544
    A=D+A
    M=0
    @16576
    A=D+A
    M=0
    @16608
    A=D+A
    M=0

    // обнуляю предыдущие смещение
    @PREVIOUS_OFFSET
    M=0

    // если текущая клавиша не UP, DOWN, LEFT, RIGHT, то иду в цикл
    @CURRENT_OFFSET
    D=M
    @LOOP
    D;JEQ
    // иначе иду рисовать
    @PAINT
    0;JMP
// Smile:
// 0001110000111000
// 0001110000111000
// 0001110000111000
// 0000000000000000
// 0000000000000000
// 0110000000000110
// 0011100000011100
// 0000111111110000

// R0 == top*32 + left/16


@16384
D = A

@0
M = M + D

@7224
D = A

@0
A = M
M = D

@32
D = A

@0
M = M + D

@7224
D = A

@0
A = M
M = D

@32
D = A

@0
M = M + D

@7224
D = A

@0
A = M
M = D


@32
D = A

@0
M = M + D

@32
D = A

@0
M = M + D

@32
D = A

@0
M = M + D

@24582
D = A

@0
A = M
M = D

@32
D = A

@0
M = M + D

@14364
D = A

@0
A = M
M = D

@0
A = M
M = D

@32
D = A

@0
M = M + D

@4080
D = A

@0
A = M
M = D
