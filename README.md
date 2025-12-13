using System;
using System.Globalization;
using Avalonia;
using Avalonia.Input;
using Avalonia.Media;

namespace Manipulation;

public static class VisualizerTask
{
    public static double X = 220;
    public static double Y = -100;
    public static double Alpha = 0.05;
    public static double Wrist = 2 * Math.PI / 3;
    public static double Elbow = 3 * Math.PI / 4;
    public static double Shoulder = Math.PI / 2;
    
    public static Brush UnreachableAreaBrush = new SolidColorBrush(Color.FromArgb(255, 255, 230, 230));
    public static Brush ReachableAreaBrush = new SolidColorBrush(Color.FromArgb(255, 230, 255, 230));
    public static Pen ManipulatorPen = new Pen(Brushes.Black, 3);
    public static Brush JointBrush = new SolidColorBrush(Colors.Gray);

    public static void KeyDown(Visual visual, KeyEventArgs key)
    {
        const double deltaAngle = 0.05;
        UpdateAngles(key.Key, deltaAngle);
        UpdateWristAngle();
        visual.InvalidateVisual();
    }

    public static void MouseMove(Visual visual, PointerEventArgs e)
    {
        UpdateCoordinates(visual, e);
        UpdateManipulator();
        visual.InvalidateVisual();
    }

    public static void MouseWheel(Visual visual, PointerWheelEventArgs e)
    {
        Alpha += e.Delta.Y * 0.05;
        UpdateManipulator();
        visual.InvalidateVisual();
    }

    public static void UpdateManipulator()
    {
        var angles = ManipulatorTask.MoveManipulatorTo(X, Y, Alpha);
        if (angles != null && angles.Length >= 3)
        {
            Shoulder = angles[0];
            Elbow = angles[1];
            Wrist = angles[2];
        }
    }

    public static void DrawManipulator(DrawingContext context, Point shoulderPos)
    {
        var joints = AnglesToCoordinatesTask.GetJointPositions(Shoulder, Elbow, Wrist);
        
        DrawReachableZone(context, shoulderPos, joints);
        DrawInfoText(context);
        DrawManipulatorLines(context, shoulderPos, joints);
        DrawJoints(context, shoulderPos, joints);
    }

    private static void UpdateAngles(Key key, double deltaAngle)
    {
        switch (key)
        {
            case Key.Q:
                Shoulder += deltaAngle;
                break;
            case Key.A:
                Shoulder -= deltaAngle;
                break;
            case Key.W:
                Elbow += deltaAngle;
                break;
            case Key.S:
                Elbow -= deltaAngle;
                break;
        }
    }

    private static void UpdateWristAngle()
    {
        Wrist = -Alpha - Shoulder - Elbow;
    }

    private static void UpdateCoordinates(Visual visual, PointerEventArgs e)
    {
        var shoulderPosition = GetShoulderPos(visual);
        var mousePosition = e.GetPosition(visual);
        var logicalCoordinates = ConvertWindowToMath(mousePosition, shoulderPosition);

        X = logicalCoordinates.X;
        Y = logicalCoordinates.Y;
    }

    private static void DrawInfoText(DrawingContext context)
    {
        var infoText = CreateInfoText();
        context.DrawText(infoText, new Point(10, 10));
    }

    private static FormattedText CreateInfoText()
    {
        return new FormattedText(
            $"X={X:0}, Y={Y:0}, Alpha={Alpha:0.00}",
            CultureInfo.InvariantCulture,
            FlowDirection.LeftToRight,
            Typeface.Default,
            18,
            Brushes.DarkRed
        )
        {
            TextAlignment = TextAlignment.Center
        };
    }

    private static void DrawManipulatorLines(DrawingContext context, Point shoulderPos, Point[] joints)
    {
        DrawArmSegments(context, shoulderPos, joints);
        DrawShoulderToArmSegment(context, shoulderPos, joints);
    }

    private static void DrawArmSegments(DrawingContext context, Point shoulderPos, Point[] joints)
    {
        for (int i = 0; i < joints.Length - 1; i++)
        {
            var startPoint = ConvertMathToWindow(joints[i], shoulderPos);
            var endPoint = ConvertMathToWindow(joints[i + 1], shoulderPos);
            context.DrawLine(ManipulatorPen, startPoint, endPoint);
        }
    }

    private static void DrawShoulderToArmSegment(DrawingContext context, Point shoulderPos, Point[] joints)
    {
        if (joints.Length > 0)
        {
            var shoulderToFirstJoint = ConvertMathToWindow(joints[0], shoulderPos);
            context.DrawLine(ManipulatorPen, shoulderPos, shoulderToFirstJoint);
        }
    }

    private static void DrawJoints(DrawingContext context, Point shoulderPos, Point[] joints)
    {
        foreach (var joint in joints)
        {
            var windowJointPosition = ConvertMathToWindow(joint, shoulderPos);
            context.DrawEllipse(JointBrush, null, windowJointPosition, 5, 5);
        }
    }

    private static void DrawReachableZone(DrawingContext context, Point shoulderPos, Point[] joints)
    {
        var rmin = Math.Abs(Manipulator.UpperArm - Manipulator.Forearm);
        var rmax = Manipulator.UpperArm + Manipulator.Forearm;
        var mathCenter = CalculateZoneCenter(joints);
        var windowCenter = ConvertMathToWindow(mathCenter, shoulderPos);
        
        DrawZoneEllipse(context, ReachableAreaBrush, windowCenter, rmax);
        DrawZoneEllipse(context, UnreachableAreaBrush, windowCenter, rmin);
    }

    private static Point CalculateZoneCenter(Point[] joints)
    {
        if (joints.Length >= 3)
            return new Point(joints[2].X - joints[1].X, joints[2].Y - joints[1].Y);
        return new Point(0, 0);
    }

    private static void DrawZoneEllipse(DrawingContext context, Brush brush, Point center, double radius)
    {
        context.DrawEllipse(brush, null, center, radius, radius);
    }

    public static Point GetShoulderPos(Visual visual)
    {
        if (visual is Avalonia.Controls.Control control)
            return new Point(control.Bounds.Width / 2, control.Bounds.Height / 2);
        return new Point(400, 300);
    }

    public static Point ConvertMathToWindow(Point mathPoint, Point shoulderPos)
    {
        return new Point(mathPoint.X + shoulderPos.X, shoulderPos.Y - mathPoint.Y);
    }

    public static Point ConvertWindowToMath(Point windowPoint, Point shoulderPos)
    {
        return new Point(windowPoint.X - shoulderPos.X, shoulderPos.Y - windowPoint.Y);
    }
}using System.Text;
namespace Assembler
{
    public class Parser
    {
        public string[] RemoveWhitespacesAndComments(string[] asmLines)
        {
            var answers = new List<string>();
            foreach (string line in asmLines)
            {
                var stringBuilder = new StringBuilder();
                var formate_line = line.Split("//");
                for (var index = 0; index < formate_line[0].Length; index++)
                    if (formate_line[0][index] != ' ')
                        stringBuilder.Append(formate_line[0][index]);
                if (!string.IsNullOrEmpty(stringBuilder.ToString()))
                    answers.Add(stringBuilder.ToString());
            }
            return answers.ToArray();
        }
    }
}

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
