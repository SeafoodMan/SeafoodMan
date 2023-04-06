<!-- URL Dictionary -->
[semiurl]: https://www.semi.org/
[semistd?]: https://www.semi.org/en/products-services/standards/using-semi-standards
[hbturl]: http://hbtechnology.co.kr/
[xcomurl]: https://www.linkgenesis.co.kr/

[E5]: https://github.com/SeafoodMan/SeafoodMan/blob/4225ed29443bfb172b4867b039938670531313c7/report/readme/docs/SEMI%20Standard%20-%20E005(SECS-II)-0704.pdf
[E37]: https://github.com/SeafoodMan/SeafoodMan/blob/4225ed29443bfb172b4867b039938670531313c7/report/readme/docs/SEMI%20Standard%20-%20E037(HSMS)-1018.pdf
[E30]: https://github.com/SeafoodMan/SeafoodMan/blob/4225ed29443bfb172b4867b039938670531313c7/report/readme/docs/SEMI%20Standard%20-%20E030(GEM)-1103.pdf

[configDev]: https://github.com/hbtechnology-SWRelated/src_HSMS-SECS/issues/45#issue-1195422557


# 주요 프로젝트

## AOI설비 INDEXER 종속형 설비 CIM 개발

## HSMS/SECS Driver 개발
### Purposes
1. SEMI Standard에서 규정한 SECS 통신 기술의 내재화
2. 상용 Driver 구매비용 절감
3. etc...

### What is SEMI Standard?
* SEMI
  + [Semiconductor Equipment and Materials International(국제 반도체 장비/재료 협회)][semiurl]
  + 세계 반도체 장비, 재료 산업 및 평판 디스플레이(FPD), MEMS, NEMS, 태양광 산업을 대표하는 세계 유일의 국제 협회
* [SEMI Standard(Series)][semistd?]
    + Standards are voluntary technical agreements between suppliers and customers, aimed at improving product quality and reliability at a reasonable price and steady supply. Standards ensure compatibility and inter-operability of goods and services. SEMI Standards are written documents and can take the form of specifications, guides, test methods, terminology, or best practices. Documents are published in a 16 volume set called SEMI International Standards.
        - M : Wafers & Process Control
        - MF : Metrology
        - T : Traceability
        - <span style="color:yellowgreen; font-family:bold; font-size:1.1em">E : Equipment communications</span>
        - A : Backend automation
        - P : Microlithography
        - S : Safety, Environmental & Energy
        - F : Facilities
        - C : Chemicals & Gases
        - G : Packaging
        - 3D : 3D Packaging
        - D : Flat panel Displays
        - MS : MEMS & NEMS
* SECS?
    + SEMI Equipment Communications Standard(E Series)
        - E4 : SECS-I(SEMI Equipment communications standard 1)
        - <span style="color:yellowgreen; font-family:bold; ">E5 : SECS-II(SEMI Equipment communications standard 2)</span>
        - <span style="color:yellowgreen; font-family:bold; ">E37 : HSMS(Hight-Speed SECS Message services)</span>
        - <span style="color:yellowgreen; font-family:bold; ">E30 : GEM(Generic model form communications and control of manufacturing equipment)</span>
        - etc...





### Development
#### Development period
* 2021.01 ~ <span style="color:yellowgreen; font-family:bold; font-size:1.5em">continuous</span>
#### Reference documents
* [E005-0699(SECS-II)][E5]
* [E037-1080(HSMS)][E37]

#### Environment
* Tool : Visual Studio 2017
* OS : Microsoft Windows
* Framework : Microsoft .Net
* Language : C#

#### Developed driver scope
<table>
    <tr align=center>
        <td><img src="/report/readme/imgs/cim_layer.png" width=360 height=360></td>
        <td><img src="/report/readme/imgs/driver_layer.png" width=360 height=360></td>
    </tr> 
    <tr align=center>
        <td><strong>Layout of equipment side SW</strong></td>
        <td><strong>Layout of Driver scope</strong></td>
    </tr>
</table>


#### Main development items
* ```TCP/IP```
  + Asynchronous TCP/IP
  + Include Active(Socket Client), Passive(Socket Server) mode
  + 1:1 Single session communication for now
```C#
public abstract class AsyncSocketV2 : _BaseComm
{
    public const int LENGTH_RECV_BUFFER = 65535;

    private const int backLog = 10;
    private readonly object locker = new object();

    private bool _IsActive = false;
    private int _Port = 8000;
    private string _ServerIP = "127.0.0.1";
    protected byte[] SendData = null;
    protected Socket _TcpActiveConnector = null;
    protected Socket _TcpListener = null;
    private CancellationTokenSource _ReconnectCanceller;

    [XmlIgnore]
    public ObservableCollection<Socket> TcpClients { get; set; } = new ObservableCollection<Socket>();

    [XmlElement]
    public bool IsActive

    protected virtual void Receive(Socket worker){}
    private async Task RunReceiveAsync(Socket sock){}

    public virtual async Task SendAsync(byte[] data, object externaldata = null){}
    public virtual Task<int> SendAsync(string data, Socket target, object externaldata = null){}
}
```


* ```HSMS, SECS-II```
  + Inheritance AsyncSocketV2(TCP/IP)
  + Include HSMS Functions, SECS-II Standard(Stream9, etc) and Logger
```C#
[Serializable]
[XmlRoot("HSMS")]
public class HSMS : AsyncSocketV2, IHaveLogger
{
    private EnConnectionState _ConnectionState = EnConnectionState.NotConnected;
    private string _SMLPath = $@"D:\sml.xml";
    private EnEntityType _EntityType = EnEntityType.Eq;
    private HTimeoutHandler _HTimeoutHandler = new HTimeoutHandler();
    private HDeviceIDHandler _HDeviceIDHandler = new HDeviceIDHandler();
    private HLoggerConfig _LoggerConfig = new HLoggerConfig();
    private SMLGroup _SMLGroup = new SMLGroup();
    private SysErrHandler _SyErrHandler = new SysErrHandler();
    private HTransactionChecker _HTransactionChecker = new HTransactionChecker();

    private BytesAccumulator _RecvBuffer = new BytesAccumulator(LENGTH_RECV_BUFFER);
    private HMessage _HMessage = null;
    private HLogger _Logger = null;

    protected HMsgBufferHandler _HMsgBufferHandler = new HMsgBufferHandler();


    [XmlElement]
    public EnEntityType EntityType

    public HSMS(int port) : this(){}

    public void Receive(HMessage message){}
    public void Send(HMessage message){}
    public int MakeSystemByte(){}
    public bool IsMessageInvalid(HMessage recvmsg){}}
}
```


* ```HSMS Header packet make method```
  + reference SEMI.E037
  + <img src="/report/readme/imgs/hsms_header.png" width=360 height=360>
```C#
public static HMessage MakeNewMessage(short devid, short stream, short func, int sysbyte, EnSECSTransactionType tratype , EnSessionCode ctrlcd = EnSessionCode.DataMessage)
{
    HMessage ret = new HMessage();

    byte[] id = devid.ToBytes();
    byte s = (byte)stream;
    byte f = (byte)func;
    byte[] sys = sysbyte.ToBytes();
    EnSessionCode ctrl = ctrlcd;

    //Device ID(Session ID)
    ret._Body.Add(id[1]);
    ret._Body.Add(id[0]);

    //Stream
    switch (ctrl)
    {
        default: ret._Body.Add(0); break;
        case EnSessionCode.DataMessage:
            {
                if (tratype == EnSECSTransactionType.Primary)
                    ret._Body.Add((byte)(s | 0x80));
                else
                    ret._Body.Add(s);
                break;
            }
        case EnSessionCode.RejectRequest:
            {
                ret._Body.Add((byte)tratype);
                break;
            }
    }

    //Functions
    switch (ctrl)
    {
        default: ret._Body.Add(0); break;
        case EnSessionCode.DataMessage:
        case EnSessionCode.SelectedResponse:
        case EnSessionCode.DeselectedResponse:
        case EnSessionCode.RejectRequest:
                ret._Body.Add(f);
                break;
    }

    //P Type(Presentation Type)
    ret._Body.Add(0);

    //S Type(Session Type)
    ret._Body.Add((byte)ctrl);

    //Systembyte
    ret._Body.Add(sys[3]);
    ret._Body.Add(sys[2]);
    ret._Body.Add(sys[1]);
    ret._Body.Add(sys[0]);

    return ret;
}
```

* ```SECSII Item format defined```
  + reference SEMI.E005
  + <img src="/report/readme/imgs/formatcode.png" width=360 height=360>
```C#
public enum EnItemFormatCode : byte
{
    [Description("LIST")]
    [XmlEnum(Name = "L")]
    LIST = 0b00000000,
    [Description("BINARY")]
    [XmlEnum(Name = "BIN")]
    Binary = 0b00100000,
    [Description("BOOL")]
    [XmlEnum(Name = "BOOL")]
    Boolean = 0b00100100,
    [Description("ASCII")]
    [XmlEnum(Name = "A")]
    ASCII = 0b01000000,
    [Description("JIS8")]
    [XmlEnum(Name = "J")]
    JIS_8 = 0b01000100, //JIS-8
    [Description("2BYTECHAR")]
    [XmlEnum(Name = "MC")]
    TwoByteChar = 0b01001000, // 2-byte character
    [Description("INT8")]
    [XmlEnum(Name = "I8")]
    I8 = 0b01100000, //8-byte signed integer
    [Description("INT1")]
    [XmlEnum(Name = "I1")]
    I1 = 0b01100100, //1-byte signed integer
    [Description("INT2")]
    [XmlEnum(Name = "I2")]
    I2 = 0b01101000, //2-byte signed integer
    [Description("INT4")]
    [XmlEnum(Name = "I4")]
    I4 = 0b01110000, //4-byte signed integer
    [Description("FLOAT8")]
    [XmlEnum(Name = "F8")]
    F8 = 0b10000000, //8-byte floating point
    [Description("FLOAT4")]
    [XmlEnum(Name = "F4")]
    F4 = 0b10010000, //4-byte floating point
    [Description("UINT8")]
    [XmlEnum(Name = "U8")]
    U8 = 0b10100000, //8-byte unsigned integer
    [Description("UINT1")]
    [XmlEnum(Name = "U1")]
    U1 = 0b10100100, //1-byte unsigned integer
    [Description("UINT2")]
    [XmlEnum(Name = "U2")]
    U2 = 0b10101000, //2-byte unsigned integer
    [Description("UINT4")]
    [XmlEnum(Name = "U4")]
    U4 = 0b10110000, //4-byte unsigned integer
}
```



* ```SECSII Item Encoder```
  + reference SEMI.E005
  + <img src="/report/readme/imgs/secs2item.png" width=360 height=360>
```C#
public unsafe void AppendItem(EnItemFormatCode code, int length, ReadOnlySpan<byte> body)
{
    bool isBodyEmpty = body.IsEmpty;
    int count = length;
    int bodyCount = 0;
    byte lenbits = 0;
    if (!isBodyEmpty)
    {
        count = body.Length;
        bodyCount = count;
    }

    byte* bytes = stackalloc byte[4];
    var countSpan = new Span<byte>(bytes, 4);
    BinaryPrimitives.WriteInt32BigEndian(countSpan, count);

    if (count <= 0xFF)
        lenbits = 1;
    else if (count <= 0xFFFF)
        lenbits = 2;
    else if (count <= 0xFFFFFF)
        lenbits = 3;

    if (lenbits > 0)
    {
        var span = GetNextWriteSpan(lenbits + 1 + bodyCount);
        span[0] = (byte)((byte)code | lenbits);
        countSpan.Slice(4 - lenbits, lenbits).CopyTo(span.Slice(1, lenbits));

        if (bodyCount > 0)
            body.CopyTo(span.Slice(lenbits + 1));

        Advance(lenbits + 1 + bodyCount);
    }
}
```


### Simulation
* Test environment

  + Sequence diagram
<table>
  <tr align=center>
      <td rowspan="4"><img src="/report/readme/imgs/sequencediagram.png" width=680 height=640></td>
      <td><img src="/report/readme/imgs/eqp_simul.png" width=480 height=220></td>
      <td>EQP_SW(Simulator)</td>
  </tr>

  <tr align=center>
      <td><img src="/report/readme/imgs/cim_sw.png" width=480 height=220></td>
      <td>CIM_SW</td>
  </tr>

  <tr align=center>
      <td><img src="/report/readme/imgs/server_gui.png" width=480 height=120></td>
      <td>Server</td>
  </tr>

  <tr align=center>
      <td><img src="/report/readme/imgs/client.PNG" width=480 height=220></td>
      <td>Client</td>
  </tr>
    
</table>



* Test Demo
![screensh](/report/readme/imgs/simulation.gif)









### Contributors
<details>
<summary><b>Team members</b></summary>

* Main members
  + <table>
      <tr align="center">
        <td>
          <a href="https://github.com/SeafoodMan">
            <img src="https://github.com/SeafoodMan.png" width="32" height="32" alt="Main members" align="center"/>
          </a>
        </td>
        <td>
          <a href="https://github.com/sym1945">
            <img src="https://github.com/sym1945.png" width="32" height="32" alt="Main members" valign="middle"/>
          </a>
        </td>
        <td>
          <a href="https://github.com/jineui-Lee">
            <img src="https://github.com/jineui-Lee.png" width="32" height="32" alt="Main members" align="center"/>
          </a>
        </td>
      </tr>

      <tr>
        <th><a href="https://github.com/SeafoodMan">지형락</a></th>
        <th><a href="https://github.com/sym1945">XXX</a></th>
        <th><a href="https://github.com/jineui-Lee">XXX</a></th>
      </tr>
    </table> 
* Support members
</details>

<details>
<summary><b>External contributors</b></summary>

* Top-tier contributors
  + none
* Other-tier contributors
  + none
</details>



### To do
#### 1. Continuous improvement work
* Bug Fix
* Improving performance and user interface
* Framework change
  + .Net Standard, .Net Framework -> .Net5

<!--
**SeafoodMan/SeafoodMan** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...

- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->
