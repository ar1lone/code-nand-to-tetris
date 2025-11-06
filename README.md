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
