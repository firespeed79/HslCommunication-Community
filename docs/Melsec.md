## Summary
��ƪ������Ҫ�������������PLC�������ݽ�����ʹ��Э��MCЭ�飬Ŀǰ֧��Qna����3E֡�Ķ�����ͨѶ��ASCIIͨѶ��
�����Э��֧֡���Ժ���˵���ں�PLCͨѶǰ����Ҫ����PLC�����������������ã����óɹ�����ܱ�֤ͨѶ�ɹ���

This article mainly explains how to interact with Mitsubishi PLC data, using the protocol MC protocol, 
currently supports Qna compatible 3E frame binary communication and ASCII communication, 
more protocol frame support will be said later, before communication with PLC, 
need PLC side Configure the network parameters. After the configuration is successful, the communication can be guaranteed.

## Contact
QQ Group : [592132877](http://shang.qq.com/wpa/qunwpa?idkey=2278cb9c2e0c04fc305c43e41acff940499a34007dfca9e83a7291e726f9c4e8)
Email: hsl200909@163.com

## Prepare
������PLC�������صĲ������ã�������������ͼƬ

Configure the relevant parameters on the Mitsubishi PLC side. Refer to the picture below

![Picture](https://raw.githubusercontent.com/dathlin/HslCommunication/master/imgs/melsec/melsec2.jpg)
![Picture](https://raw.githubusercontent.com/dathlin/HslCommunication/master/imgs/melsec/melsec1.jpg)


## Instantiation
1. Add namespace
<pre>
<code>
using HslCommunication.Profinet.Melsec;
using HslCommunication;
</code>
</pre>

2. Declare
<pre>
<code>
private MelsecMcNet melsec_net = null;
</code>
</pre>

3. Instantiation
<pre>
<code>
 // specify plc ip address and port
 melsec_net = new MelsecMcNet("192.168.0.110",2000);
 melsec_net.ConnectServer();
</code>
</pre>

If we want to know whether connectd, we can do like this
<pre>
<code>
OperateResult connect = melsec_net.ConnectServer( );
if (connect.IsSuccess)
{
    // success
}
else
{
    // failed
}
</code>
</pre>

4. Closed when exsis programe
<pre>
<code>
 melsec_net.ConnectClose( );
</code>
</pre>

## Exchange Data

Show some examples read from plc and write into plc

1. Read M100 to M109
<pre>
<code>
OperateResult<bool[]> read = melsec_net.ReadBool( "M100", 10 );
if(read.IsSuccess)
{
    bool m100 = read.Content[0];
    // and so on
    bool m109 = read.Content[9];
}
else
{
    // failed
}
</code>
</pre>

2. Write M100 to M109
<pre>
<code>
bool[] values = new bool[] { true, false, true, true, false, true, false, true, true, false };
OperateResult read = melsec_net.Write( "M100", values );
if (read.IsSuccess)
{
    // success
}
else
{
    // failed
}
</code>
</pre>

Not only  'M' dataType, but also 'X','Y','L','F','V','B','S' 

3. Read D100 Include Many Data Types
<pre>
<code>
short d100_short = melsec_net.ReadInt16( "D100" ).Content;
ushort d100_ushort = melsec_net.ReadUInt16( "D100" ).Content;
int d100_int = melsec_net.ReadInt32( "D100" ).Content;
uint d100_uint = melsec_net.ReadUInt32( "D100" ).Content;
long d100_long = melsec_net.ReadInt64( "D100" ).Content;
ulong d100_ulong = melsec_net.ReadUInt64( "D100" ).Content;
float d100_float = melsec_net.ReadFloat( "D100" ).Content;
double d100_double = melsec_net.ReadDouble( "D100" ).Content;
// need to specify the text length
string d100_string = melsec_net.ReadString( "D100", 10 ).Content;
</code>
</pre>

4. Write D100 Include Many Data Types
<pre>
<code>
melsec_net.Write( "D100", (short)5 );
melsec_net.Write( "D100", (ushort)5 );
melsec_net.Write( "D100", 5 );
melsec_net.Write( "D100", (uint)5 );
melsec_net.Write( "D100", (long)5 );
melsec_net.Write( "D100", (ulong)5 );
melsec_net.Write( "D100", 5f );
melsec_net.Write( "D100", 5d );
// length should Multiples of 2 
melsec_net.Write( "D100", "12345678" );
</code>
</pre>

Not only  'D' dataType, but also 'D','W','R'

This library alse support write array values.


5. Read complex data, for example, D100-D109 contains all data you want

Data name | Data section | Data type | Data Length
-|-|-|-
count | D100-D101 | int | 4-byte
temp | D102-D103 | float | 4-byte
name1 | D104 | short | 2-byte
barcode | D105-D109 | string | 10-byte

So we can do like this

```

OperateResult<byte[]> read = melsec_net.Read( "D100", 10 );
if(read.IsSuccess)
{
    int count = melsec_net.ByteTransform.TransInt32( read.Content, 0 );
    float temp = melsec_net.ByteTransform.TransSingle( read.Content, 4 );
    short name1 = melsec_net.ByteTransform.TransInt16( read.Content, 8 );
    string barcode = Encoding.ASCII.GetString( read.Content, 10, 10 );
}

```

6. Implementing custom type reads and writes

We found the code above is awkward and we want to improve.

First, Inherit and implement interface methods

```

public class UserType : HslCommunication.IDataTransfer
{
    #region IDataTransfer

    private HslCommunication.Core.IByteTransform ByteTransform = new HslCommunication.Core.RegularByteTransform();


    public ushort ReadCount => 10;

    public void ParseSource( byte[] Content )
    {
        int count = ByteTransform.TransInt32( Content, 0 );
        float temp = ByteTransform.TransSingle( Content, 4 );
        short name1 = ByteTransform.TransInt16( Content, 8 );
        string barcode = Encoding.ASCII.GetString( Content, 10, 10 );
    }

    public byte[] ToSource( )
    {
        byte[] buffer = new byte[20];
        ByteTransform.TransByte( count ).CopyTo( buffer, 0 );
        ByteTransform.TransByte( temp ).CopyTo( buffer, 4 );
        ByteTransform.TransByte( name1 ).CopyTo( buffer, 8 );
        Encoding.ASCII.GetBytes( barcode ).CopyTo( buffer, 10 );
        return buffer;
    }


    #endregion


    #region Public Data

    public int count { get; set; }

    public float temp { get; set; }

    public short name1 { get; set; }

    public string barcode { get; set; }

    #endregion
}

```

So we can do like this

```

OperateResult<UserType> read = melsec_net.ReadCustomer<UserType>( "D100" );
if (read.IsSuccess)
{
    UserType value = read.Content;
}


```