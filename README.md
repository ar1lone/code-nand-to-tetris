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
