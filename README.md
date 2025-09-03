# MES-System

跳至內容
聊天歷程紀錄

你說：
依據runcard的流程 還有途中可能會遇到工程變更與通知 幫我根據northwind的資料庫設計一套C#網頁版的MES程式 bom tree的建立以及routing tree的建立 還有後續如何訪問bom tree 與 routing tree
ChatGPT 說：

已思考 1m 13s
太好了！下面給你一套「以 Northwind 為基底」的 C#（ASP.NET Core MVC + EF Core）網頁版 MES 參考實作：涵蓋 BOM Tree 建立/查詢、Routing Tree 建立/查詢、Runcard 流程，以及 工程變更（ECO/ECN）與通知 的模型、流程與程式碼骨架。你可以把它直接當作起始專案的雛形，逐步充實細節。

一、整體架構（概念）
DB：沿用 Northwind（Products、Suppliers…），新增 MES 所需表（BomHeader/BomLine、RoutingHeader/RoutingOperation、WorkCenter、Uom、WorkOrder/Runcard…）

服務層：

BomService：建立/讀取 BOM Tree、BOM 展開（Explosion）

RoutingService：建立/讀取 Routing、產生 Runcard Steps

EcoService：ECO 申請→審核→生效、生成通知、影響分析（對既有工單的處置）

Web 層（MVC）：

BOM：/Bom/{productId}?rev=... 顯示樹

Routing：/Routing/{productId}?rev=... 顯示製程清單

WO/Runcard：/WorkOrders/{id} 顯示 traveler（作業步驟，開工/完工/量測/簽核）

ECO：/Eco/{id} 審核與發布

通知：以資料表記錄，UI 角落顯示待辦/變更；若要即時可加 SignalR。

版本與凍結：工單發放時把 BOM 與 Routing 凍結成快照，以便「中途工程變更」不影響已發的 runcard（除非你要指定「立即生效並強制切換」）。

二、資料庫設計（DDL 範例）
下方 DDL 以 dbo schema，和 Northwind 共存。依你先前訊息，已存在 BomLine/RoutingOperation/WorkCenter/Uom 可調整欄位名稱以對齊。

-- Work Centers & UoM
CREATE TABLE dbo.WorkCenter (
  Id            INT IDENTITY PRIMARY KEY,
  Code          NVARCHAR(50) UNIQUE NOT NULL,
  Name          NVARCHAR(200) NOT NULL,
  Location      NVARCHAR(200) NULL,
  IsActive      BIT NOT NULL DEFAULT 1
);

CREATE TABLE dbo.Uom (
  Id            INT IDENTITY PRIMARY KEY,
  Code          NVARCHAR(20) UNIQUE NOT NULL,
  Description   NVARCHAR(100) NULL
);

-- BOM（支援版本/生效區間與階層）
CREATE TABLE dbo.BomHeader (
  Id             INT IDENTITY PRIMARY KEY,
  ProductId      INT NOT NULL,          -- 對應 Northwind.dbo.Products.ProductID
  Revision       NVARCHAR(20) NOT NULL, -- 例：A、B、C 或 1.0/1.1
  IsReleased     BIT NOT NULL DEFAULT 0,
  EffectiveFrom  DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
  EffectiveTo    DATETIME2 NULL,        -- NULL=仍有效
  Status         NVARCHAR(20) NOT NULL DEFAULT N'Draft', -- Draft/Released/Obsolete
  CONSTRAINT UX_BomHeader UNIQUE (ProductId, Revision)
);

CREATE TABLE dbo.BomLine (
  Id               INT IDENTITY PRIMARY KEY,
  BomHeaderId      INT NOT NULL,
  -- 階層化：以 ParentLineId 表示父階
  ParentLineId     INT NULL,
  LineNo           INT NOT NULL,
  ComponentProductId INT NOT NULL,      -- 對應 Products.ProductID
  QtyPer           DECIMAL(18,6) NOT NULL,
  UomId            INT NULL,
  ScrapFactorPct   DECIMAL(9,6) NULL,   -- 報廢率%
  IsOptional       BIT NOT NULL DEFAULT 0,
  RefDes           NVARCHAR(200) NULL,  -- 參考標示/位號
  Notes            NVARCHAR(500) NULL,
  CONSTRAINT FK_BomLine_Header FOREIGN KEY (BomHeaderId) REFERENCES dbo.BomHeader(Id),
  CONSTRAINT FK_BomLine_Parent FOREIGN KEY (ParentLineId) REFERENCES dbo.BomLine(Id),
  CONSTRAINT FK_BomLine_Uom FOREIGN KEY (UomId) REFERENCES dbo.Uom(Id)
);
CREATE INDEX IX_BomLine_BomHeader ON dbo.BomLine(BomHeaderId);
CREATE INDEX IX_BomLine_Parent ON dbo.BomLine(ParentLineId);

-- Routing（支援版本/生效/工序順序）
CREATE TABLE dbo.RoutingHeader (
  Id             INT IDENTITY PRIMARY KEY,
  ProductId      INT NOT NULL,
  Revision       NVARCHAR(20) NOT NULL,
  IsReleased     BIT NOT NULL DEFAULT 0,
  EffectiveFrom  DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
  EffectiveTo    DATETIME2 NULL,
  Status         NVARCHAR(20) NOT NULL DEFAULT N'Draft',
  CONSTRAINT UX_RoutingHeader UNIQUE (ProductId, Revision)
);

CREATE TABLE dbo.RoutingOperation (
  Id              INT IDENTITY PRIMARY KEY,
  RoutingHeaderId INT NOT NULL,
  Seq             INT NOT NULL,               -- 工序順序
  WorkCenterId    INT NOT NULL,
  OperationCode   NVARCHAR(50) NOT NULL,
  StdSetupTimeMin DECIMAL(18,4) NULL,
  StdRunTimeMin   DECIMAL(18,4) NULL,
  UomId           INT NULL,                   -- 生產速率單位
  Instruction     NVARCHAR(MAX) NULL,
  CheckpointsJson NVARCHAR(MAX) NULL,         -- 檢查點/量測項 JSON
  CONSTRAINT FK_RoutingOperation_Header FOREIGN KEY (RoutingHeaderId) REFERENCES dbo.RoutingHeader(Id),
  CONSTRAINT FK_RoutingOperation_WC FOREIGN KEY (WorkCenterId) REFERENCES dbo.WorkCenter(Id),
  CONSTRAINT FK_RoutingOperation_Uom FOREIGN KEY (UomId) REFERENCES dbo.Uom(Id)
);
CREATE INDEX IX_RoutingOperation_HeaderSeq ON dbo.RoutingOperation(RoutingHeaderId, Seq);

-- 工單與 Runcard（凍結快照）
CREATE TABLE dbo.WorkOrder (
  Id                 INT IDENTITY PRIMARY KEY,
  Number             NVARCHAR(50) UNIQUE NOT NULL,
  ProductId          INT NOT NULL,
  Qty                DECIMAL(18,6) NOT NULL,
  DueDate            DATE NULL,
  BomRevisionFrozen  NVARCHAR(20) NOT NULL,
  RoutingRevisionFrozen NVARCHAR(20) NOT NULL,
  CreatedAt          DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
  Status             NVARCHAR(20) NOT NULL DEFAULT N'Released'  -- Released/InProgress/Completed/Closed/Hold
);

CREATE TABLE dbo.WorkOrderBomSnapshot (
  Id                 INT IDENTITY PRIMARY KEY,
  WorkOrderId        INT NOT NULL,
  LineNo             INT NOT NULL,
  ParentLineNo       INT NULL,             -- 用凍結後的 LineNo 關聯
  ComponentProductId INT NOT NULL,
  QtyPer             DECIMAL(18,6) NOT NULL,
  UomCode            NVARCHAR(20) NULL,
  Notes              NVARCHAR(500) NULL,
  CONSTRAINT FK_WOBS_WO FOREIGN KEY (WorkOrderId) REFERENCES dbo.WorkOrder(Id)
);

CREATE TABLE dbo.WorkOrderRoutingSnapshot (
  Id                 INT IDENTITY PRIMARY KEY,
  WorkOrderId        INT NOT NULL,
  Seq                INT NOT NULL,
  WorkCenterCode     NVARCHAR(50) NOT NULL,
  OperationCode      NVARCHAR(50) NOT NULL,
  StdSetupTimeMin    DECIMAL(18,4) NULL,
  StdRunTimeMin      DECIMAL(18,4) NULL,
  Instruction        NVARCHAR(MAX) NULL,
  CheckpointsJson    NVARCHAR(MAX) NULL,
  CONSTRAINT FK_WORS_WO FOREIGN KEY (WorkOrderId) REFERENCES dbo.WorkOrder(Id)
);

CREATE TABLE dbo.Runcard (
  Id                 INT IDENTITY PRIMARY KEY,
  WorkOrderId        INT NOT NULL,
  LotNo              NVARCHAR(50) NOT NULL,
  Status             NVARCHAR(20) NOT NULL DEFAULT N'Open', -- Open/Running/Done/Scrapped/Hold
  CreatedAt          DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
  CONSTRAINT UX_Runcard UNIQUE (WorkOrderId, LotNo),
  CONSTRAINT FK_Runcard_WO FOREIGN KEY (WorkOrderId) REFERENCES dbo.WorkOrder(Id)
);

CREATE TABLE dbo.RuncardStep (
  Id                 INT IDENTITY PRIMARY KEY,
  RuncardId          INT NOT NULL,
  Seq                INT NOT NULL,
  WorkCenterCode     NVARCHAR(50) NOT NULL,
  OperationCode      NVARCHAR(50) NOT NULL,
  StartTime          DATETIME2 NULL,
  EndTime            DATETIME2 NULL,
  Status             NVARCHAR(20) NOT NULL DEFAULT N'Pending', -- Pending/Processing/Completed/Hold/NCR
  OperatorUserId     NVARCHAR(50) NULL,
  MeasurementsJson   NVARCHAR(MAX) NULL, -- 量測/檢驗記錄
  DefectsJson        NVARCHAR(MAX) NULL,
  SignOff            NVARCHAR(200) NULL,
  CONSTRAINT UX_RuncardStep UNIQUE (RuncardId, Seq),
  CONSTRAINT FK_RuncardStep_Runcard FOREIGN KEY (RuncardId) REFERENCES dbo.Runcard(Id)
);

-- 工程變更（ECO/ECN）與通知
CREATE TABLE dbo.ECO (
  Id              INT IDENTITY PRIMARY KEY,
  Number          NVARCHAR(50) UNIQUE NOT NULL,
  Type            NVARCHAR(20) NOT NULL,      -- BOM/Routing/Both
  TargetProductId INT NOT NULL,
  FromRevision    NVARCHAR(20) NULL,
  ToRevision      NVARCHAR(20) NOT NULL,
  EffectivityType NVARCHAR(20) NOT NULL,      -- Date/Lot/WO/Revision
  EffectiveFrom   DATETIME2 NULL,             -- Date 型用
  Status          NVARCHAR(20) NOT NULL DEFAULT N'Draft', -- Draft/Approved/Released/Cancelled
  Reason          NVARCHAR(500) NULL,
  CreatedBy       NVARCHAR(50) NULL,
  CreatedAt       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE TABLE dbo.ECODetail (
  Id            INT IDENTITY PRIMARY KEY,
  ECOId         INT NOT NULL,
  Target        NVARCHAR(20) NOT NULL,   -- BOMLine/RoutingOperation
  Action        NVARCHAR(10) NOT NULL,   -- Add/Update/Remove
  RefId         INT NULL,                -- 針對既有項目的 Id
  PatchJson     NVARCHAR(MAX) NULL,      -- 變更內容（例如 QtyPer:1.2→1.1）
  CONSTRAINT FK_ECODetail_ECO FOREIGN KEY (ECOId) REFERENCES dbo.ECO(Id)
);

CREATE TABLE dbo.ECNNotification (
  Id            INT IDENTITY PRIMARY KEY,
  ECOId         INT NOT NULL,
  ToUserId      NVARCHAR(50) NOT NULL,
  Channel       NVARCHAR(20) NOT NULL,   -- UI/Email/Slack…
  SentAt        DATETIME2 NULL,
  AcknowledgedAt DATETIME2 NULL,
  CONSTRAINT FK_ECN_ECO FOREIGN KEY (ECOId) REFERENCES dbo.ECO(Id)
);
效能/錯誤提醒：

為 BomLine(BomHeaderId, ParentLineId)、RoutingOperation(RoutingHeaderId, Seq) 建索引（上面已建）。

BOM Explosion 用 CTE + MAXRECURSION，避免環狀；對大 BOM 建 path 索引或預算表（展開快照）。

你曾遇到 SQL 記憶體 701，請避免一次性展開過大集合、分批處理或做「每工單展開快照」以減壓。

三、EF Core 模型（精簡）
// 對應 Northwind.Products（只取必要欄位）
public class Product {
    public int ProductID { get; set; }
    public string ProductName { get; set; } = "";
}

// 基本 MES 實體（部分）
public class BomHeader {
    public int Id { get; set; }
    public int ProductId { get; set; }
    public string Revision { get; set; } = "A";
    public bool IsReleased { get; set; }
    public DateTime EffectiveFrom { get; set; }
    public DateTime? EffectiveTo { get; set; }
    public string Status { get; set; } = "Draft";
    public List<BomLine> Lines { get; set; } = new();
}

public class BomLine {
    public int Id { get; set; }
    public int BomHeaderId { get; set; }
    public int? ParentLineId { get; set; }
    public int LineNo { get; set; }
    public int ComponentProductId { get; set; }
    public decimal QtyPer { get; set; }
    public int? UomId { get; set; }
    public decimal? ScrapFactorPct { get; set; }
    public bool IsOptional { get; set; }
    public string? RefDes { get; set; }
    public string? Notes { get; set; }
}

public class RoutingHeader {
    public int Id { get; set; }
    public int ProductId { get; set; }
    public string Revision { get; set; } = "A";
    public bool IsReleased { get; set; }
    public DateTime EffectiveFrom { get; set; }
    public DateTime? EffectiveTo { get; set; }
    public string Status { get; set; } = "Draft";
    public List<RoutingOperation> Operations { get; set; } = new();
}

public class RoutingOperation {
    public int Id { get; set; }
    public int RoutingHeaderId { get; set; }
    public int Seq { get; set; }
    public int WorkCenterId { get; set; }
    public string OperationCode { get; set; } = "";
    public decimal? StdSetupTimeMin { get; set; }
    public decimal? StdRunTimeMin { get; set; }
    public int? UomId { get; set; }
    public string? Instruction { get; set; }
    public string? CheckpointsJson { get; set; }
}

public class WorkOrder {
    public int Id { get; set; }
    public string Number { get; set; } = "";
    public int ProductId { get; set; }
    public decimal Qty { get; set; }
    public DateTime? DueDate { get; set; }
    public string BomRevisionFrozen { get; set; } = "";
    public string RoutingRevisionFrozen { get; set; } = "";
    public DateTime CreatedAt { get; set; }
    public string Status { get; set; } = "Released";
}
DbContext（重點片段）

public class MesDbContext : DbContext
{
    public MesDbContext(DbContextOptions<MesDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>(); // map 到 Northwind
    public DbSet<BomHeader> BomHeaders => Set<BomHeader>();
    public DbSet<BomLine> BomLines => Set<BomLine>();
    public DbSet<RoutingHeader> RoutingHeaders => Set<RoutingHeader>();
    public DbSet<RoutingOperation> RoutingOperations => Set<RoutingOperation>();
    public DbSet<WorkOrder> WorkOrders => Set<WorkOrder>();
    public DbSet<WorkOrderBomSnapshot> WorkOrderBomSnapshots => Set<WorkOrderBomSnapshot>();
    public DbSet<WorkOrderRoutingSnapshot> WorkOrderRoutingSnapshots => Set<WorkOrderRoutingSnapshot>();
    public DbSet<Runcard> Runcards => Set<Runcard>();
    public DbSet<RuncardStep> RuncardSteps => Set<RuncardStep>();
    public DbSet<ECO> ECOs => Set<ECO>();
    public DbSet<ECODetail> ECODetails => Set<ECODetail>();
    public DbSet<ECNNotification> ECNNotifications => Set<ECNNotification>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        // Northwind Products
        b.Entity<Product>().ToTable("Products", "dbo").HasKey(x => x.ProductID);
        b.Entity<Product>().Property(x => x.ProductName).HasColumnType("nvarchar(40)");

        // 其餘表用預設對應 dbo.*
        b.Entity<BomHeader>().HasIndex(x => new { x.ProductId, x.Revision }).IsUnique();

        b.Entity<BomLine>()
          .HasOne<BomLine>()
          .WithMany()
          .HasForeignKey(x => x.ParentLineId)
          .OnDelete(DeleteBehavior.Restrict);

        b.Entity<RoutingHeader>().HasIndex(x => new { x.ProductId, x.Revision }).IsUnique();
        b.Entity<RoutingOperation>().HasIndex(x => new { x.RoutingHeaderId, x.Seq }).IsUnique();

        // Snapshot 唯一鍵
        b.Entity<WorkOrderRoutingSnapshot>().HasIndex(x => new { x.WorkOrderId, x.Seq }).IsUnique();
        b.Entity<RuncardStep>().HasIndex(x => new { x.RuncardId, x.Seq }).IsUnique();
    }
}
Startup.cs（.NET 6 亦可保留 Startup 模式）

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllersWithViews();
        services.AddDbContext<MesDbContext>(opt =>
            opt.UseSqlServer(Configuration.GetConnectionString("NorthwindMes")));
        services.AddScoped<BomService>();
        services.AddScoped<RoutingService>();
        services.AddScoped<EcoService>();
        services.AddScoped<WorkOrderService>();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment()) app.UseDeveloperExceptionPage();

        app.UseStaticFiles();
        app.UseRouting();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllerRoute(
                name: "default",
                pattern: "{controller=Home}/{action=Index}/{id?}");
        });
    }
}
四、BOM Tree：建立與查詢
1) 建立（Controller 範例）
[Route("api/bom")]
public class BomController : Controller
{
    private readonly BomService _bom;

    public BomController(BomService bom) => _bom = bom;

    // 建立或更新一版 BOM（支援階層）
    [HttpPost]
    public async Task<IActionResult> Upsert([FromBody] UpsertBomRequest req)
    {
        var id = await _bom.UpsertAsync(req);
        return Ok(new { BomHeaderId = id });
    }

    // 以產品 + 版號（或 AsOf 時點自動挑有效版）取 BOM Tree
    [HttpGet("{productId}")]
    public async Task<IActionResult> GetTree(int productId, [FromQuery] string? rev, [FromQuery] DateTime? asOf)
    {
        var tree = await _bom.GetTreeAsync(productId, rev, asOf ?? DateTime.UtcNow);
        return Ok(tree);
    }
}

// DTO（簡化）
public record UpsertBomRequest(
    int ProductId,
    string Revision,
    bool IsReleased,
    IEnumerable<BomNodeDto> Lines
);

public record BomNodeDto(
    int LineNo,
    int ComponentProductId,
    decimal QtyPer,
    int? UomId,
    string? Notes,
    IEnumerable<BomNodeDto>? Children
);
2) 服務層（核心邏輯：階層寫入與讀取）
public class BomService
{
    private readonly MesDbContext _db;
    public BomService(MesDbContext db) => _db = db;

    public async Task<int> UpsertAsync(UpsertBomRequest req)
    {
        var header = await _db.BomHeaders
            .FirstOrDefaultAsync(x => x.ProductId == req.ProductId && x.Revision == req.Revision);

        if (header == null)
        {
            header = new BomHeader {
                ProductId = req.ProductId, Revision = req.Revision,
                IsReleased = req.IsReleased, Status = req.IsReleased ? "Released" : "Draft",
                EffectiveFrom = DateTime.UtcNow
            };
            _db.BomHeaders.Add(header);
            await _db.SaveChangesAsync();
        }
        else
        {
            // 簡化：先刪後建（真實系統可改差異補丁）
            var oldLines = _db.BomLines.Where(x => x.BomHeaderId == header.Id);
            _db.BomLines.RemoveRange(oldLines);
            await _db.SaveChangesAsync();
            header.IsReleased = req.IsReleased;
            header.Status = req.IsReleased ? "Released" : "Draft";
        }

        // 以 DFS 方式遞迴寫入
        var tempLineIdByLineNo = new Dictionary<int, int>();
        await InsertLinesRecursive(header.Id, null, req.Lines, tempLineIdByLineNo);

        await _db.SaveChangesAsync();
        return header.Id;
    }

    private async Task InsertLinesRecursive(int bomHeaderId, int? parentLineId, IEnumerable<BomNodeDto> nodes, Dictionary<int,int> dict)
    {
        foreach (var n in nodes)
        {
            var line = new BomLine {
                BomHeaderId = bomHeaderId,
                ParentLineId = parentLineId,
                LineNo = n.LineNo,
                ComponentProductId = n.ComponentProductId,
                QtyPer = n.QtyPer,
                UomId = n.UomId,
                Notes = n.Notes
            };
            _db.BomLines.Add(line);
            await _db.SaveChangesAsync();
            dict[n.LineNo] = line.Id;

            if (n.Children?.Any() == true)
                await InsertLinesRecursive(bomHeaderId, line.Id, n.Children, dict);
        }
    }

    // 讀取樹（一次抓出所有線，於記憶體拼樹；規模大可改 CTE）
    public async Task<BomTreeVm> GetTreeAsync(int productId, string? rev, DateTime asOf)
    {
        var header = await PickEffectiveBomHeader(productId, rev, asOf);
        if (header == null) throw new InvalidOperationException("No effective BOM.");

        var lines = await _db.BomLines
            .Where(x => x.BomHeaderId == header.Id)
            .OrderBy(x => x.ParentLineId).ThenBy(x => x.LineNo)
            .ToListAsync();

        var productNames = await _db.Products
            .Where(p => lines.Select(l => l.ComponentProductId).Contains(p.ProductID) || p.ProductID == productId)
            .ToDictionaryAsync(p => p.ProductID, p => p.ProductName);

        var lookup = lines.ToLookup(x => x.ParentLineId);
        Func<int?, List<BomNodeVm>> build = null!;
        build = parentId => lookup[parentId].Select(l => new BomNodeVm {
            LineNo = l.LineNo,
            ProductId = l.ComponentProductId,
            ProductName = productNames.TryGetValue(l.ComponentProductId, out var nm) ? nm : "",
            QtyPer = l.QtyPer,
            Children = build(l.Id)
        }).ToList();

        return new BomTreeVm {
            ProductId = productId,
            ProductName = productNames[productId],
            Revision = header.Revision,
            Nodes = build(null)
        };
    }

    private async Task<BomHeader?> PickEffectiveBomHeader(int productId, string? rev, DateTime asOf)
    {
        if (!string.IsNullOrWhiteSpace(rev))
            return await _db.BomHeaders.FirstOrDefaultAsync(x =>
                x.ProductId == productId && x.Revision == rev && x.IsReleased &&
                x.EffectiveFrom <= asOf && (x.EffectiveTo == null || x.EffectiveTo > asOf));

        return await _db.BomHeaders
            .Where(x => x.ProductId == productId && x.IsReleased && x.EffectiveFrom <= asOf &&
                        (x.EffectiveTo == null || x.EffectiveTo > asOf))
            .OrderByDescending(x => x.EffectiveFrom)  // 最近生效版
            .FirstOrDefaultAsync();
    }
}

// ViewModel
public record BomTreeVm(int ProductId, string ProductName, string Revision, List<BomNodeVm> Nodes)
{
    public BomTreeVm() : this(0,"","",new()) {}
}
public class BomNodeVm {
    public int LineNo { get; set; }
    public int ProductId { get; set; }
    public string ProductName { get; set; } = "";
    public decimal QtyPer { get; set; }
    public List<BomNodeVm> Children { get; set; } = new();
}
3) SQL：BOM 展開（Explosion, 多階乘數）
當你要算投入總量，可用 CTE（以 ParentLineId 階層為例）：

DECLARE @BomHeaderId INT = @pBomHeaderId;
DECLARE @TopQty DECIMAL(18,6) = @pTopQty; -- 工單總數量

;WITH CTE AS (
  SELECT 
    bl.Id, bl.ParentLineId, bl.LineNo, bl.ComponentProductId,
    CAST(bl.QtyPer AS DECIMAL(38,12)) AS QtyEach,
    CAST(bl.QtyPer AS DECIMAL(38,12)) AS CumQty,
    0 AS Lvl
  FROM dbo.BomLine bl
  WHERE bl.BomHeaderId = @BomHeaderId AND bl.ParentLineId IS NULL

  UNION ALL

  SELECT 
    c2.Id, c2.ParentLineId, c2.LineNo, c2.ComponentProductId,
    CAST(c2.QtyPer AS DECIMAL(38,12)),
    CAST(c.CumQty * c2.QtyPer AS DECIMAL(38,12)),
    c.Lvl + 1
  FROM dbo.BomLine c2
  JOIN CTE c ON c2.ParentLineId = c.Id
)
SELECT 
  c.Lvl, c.LineNo, c.ComponentProductId, p.ProductName,
  c.QtyEach, c.CumQty, 
  RequiredForOrder = c.CumQty * @TopQty
FROM CTE c
JOIN dbo.Products p ON p.ProductID = c.ComponentProductId
ORDER BY c.Lvl, c.LineNo
OPTION (MAXRECURSION 100);
五、Routing：建立與查詢
RoutingService（摘要）

public class RoutingService
{
    private readonly MesDbContext _db;
    public RoutingService(MesDbContext db) => _db = db;

    public async Task<int> UpsertAsync(int productId, string revision, IEnumerable<RoutingOpDto> ops, bool release)
    {
        var header = await _db.RoutingHeaders
            .FirstOrDefaultAsync(x => x.ProductId == productId && x.Revision == revision);

        if (header == null) {
            header = new RoutingHeader {
                ProductId = productId, Revision = revision,
                IsReleased = release, Status = release ? "Released" : "Draft",
                EffectiveFrom = DateTime.UtcNow
            };
            _db.RoutingHeaders.Add(header);
            await _db.SaveChangesAsync();
        } else {
            var oldOps = _db.RoutingOperations.Where(x => x.RoutingHeaderId == header.Id);
            _db.RoutingOperations.RemoveRange(oldOps);
            await _db.SaveChangesAsync();
            header.IsReleased = release;
            header.Status = release ? "Released" : "Draft";
        }

        int seq = 10;
        foreach (var op in ops)
        {
            _db.RoutingOperations.Add(new RoutingOperation {
                RoutingHeaderId = header.Id,
                Seq = op.Seq ?? (seq += 10),
                WorkCenterId = op.WorkCenterId,
                OperationCode = op.OperationCode,
                StdSetupTimeMin = op.StdSetupTimeMin,
                StdRunTimeMin = op.StdRunTimeMin,
                UomId = op.UomId,
                Instruction = op.Instruction,
                CheckpointsJson = op.CheckpointsJson
            });
        }
        await _db.SaveChangesAsync();
        return header.Id;
    }

    public async Task<RoutingVm> GetAsync(int productId, string? rev, DateTime asOf)
    {
        var hdr = await PickEffectiveHeader(productId, rev, asOf);
        if (hdr == null) throw new InvalidOperationException("No effective routing.");

        var ops = await _db.RoutingOperations
            .Where(x => x.RoutingHeaderId == hdr.Id)
            .OrderBy(x => x.Seq)
            .Select(x => new RoutingOpVm(
                x.Seq, x.OperationCode, x.WorkCenterId, x.StdSetupTimeMin, x.StdRunTimeMin, x.Instruction))
            .ToListAsync();

        return new RoutingVm(productId, hdr.Revision, ops);
    }

    private Task<RoutingHeader?> PickEffectiveHeader(int productId, string? rev, DateTime asOf)
    {
        if (!string.IsNullOrWhiteSpace(rev))
            return _db.RoutingHeaders.FirstOrDefaultAsync(x =>
                x.ProductId == productId && x.Revision == rev && x.IsReleased &&
                x.EffectiveFrom <= asOf && (x.EffectiveTo == null || x.EffectiveTo > asOf));

        return _db.RoutingHeaders
            .Where(x => x.ProductId == productId && x.IsReleased && x.EffectiveFrom <= asOf &&
                        (x.EffectiveTo == null || x.EffectiveTo > asOf))
            .OrderByDescending(x => x.EffectiveFrom)
            .FirstOrDefaultAsync();
    }
}

public record RoutingOpDto(int? Seq, int WorkCenterId, string OperationCode,
    decimal? StdSetupTimeMin, decimal? StdRunTimeMin, int? UomId, string? Instruction, string? CheckpointsJson);

public record RoutingVm(int ProductId, string Revision, List<RoutingOpVm> Operations);
public record RoutingOpVm(int Seq, string OperationCode, int WorkCenterId,
    decimal? StdSetupTimeMin, decimal? StdRunTimeMin, string? Instruction);
Controller：

[Route("api/routing")]
public class RoutingController : Controller
{
    private readonly RoutingService _svc;
    public RoutingController(RoutingService svc) => _svc = svc;

    [HttpPost]
    public async Task<IActionResult> Upsert([FromBody] UpsertRoutingRequest req)
    {
        var id = await _svc.UpsertAsync(req.ProductId, req.Revision, req.Operations, req.IsReleased);
        return Ok(new { RoutingHeaderId = id });
    }

    [HttpGet("{productId}")]
    public async Task<IActionResult> Get(int productId, [FromQuery] string? rev, [FromQuery] DateTime? asOf)
    {
        var vm = await _svc.GetAsync(productId, rev, asOf ?? DateTime.UtcNow);
        return Ok(vm);
    }
}

public record UpsertRoutingRequest(int ProductId, string Revision, bool IsReleased, List<RoutingOpDto> Operations);
六、Runcard（Traveler）流程
工單發放（凍結 + 產生 Runcard + 步驟）

public class WorkOrderService
{
    private readonly MesDbContext _db;
    private readonly BomService _bom;
    private readonly RoutingService _routing;

    public WorkOrderService(MesDbContext db, BomService bom, RoutingService routing)
    {
        _db = db; _bom = bom; _routing = routing;
    }

    public async Task<int> ReleaseWorkOrderAsync(string woNo, int productId, decimal qty, string? bomRev, string? routingRev)
    {
        // 1) 挑版 & 凍結
        var bomHeader = await _db.BomHeaders.FirstAsync(x => x.ProductId == productId && (bomRev == null || x.Revision == bomRev) && x.IsReleased);
        var routingHeader = await _db.RoutingHeaders.FirstAsync(x => x.ProductId == productId && (routingRev == null || x.Revision == routingRev) && x.IsReleased);

        var wo = new WorkOrder {
            Number = woNo,
            ProductId = productId,
            Qty = qty,
            BomRevisionFrozen = bomHeader.Revision,
            RoutingRevisionFrozen = routingHeader.Revision,
            CreatedAt = DateTime.UtcNow,
            Status = "Released"
        };
        _db.WorkOrders.Add(wo);
        await _db.SaveChangesAsync();

        // 2) 建立 BOM 快照
        var lines = await _db.BomLines.Where(x => x.BomHeaderId == bomHeader.Id).ToListAsync();
        var uoms = await _db.Uoms.ToDictionaryAsync(u => u.Id, u => u.Code);
        var snapshots = lines.Select(l => new WorkOrderBomSnapshot {
            WorkOrderId = wo.Id,
            LineNo = l.LineNo,
            ParentLineNo = lines.FirstOrDefault(p => p.Id == l.ParentLineId)?.LineNo,
            ComponentProductId = l.ComponentProductId,
            QtyPer = l.QtyPer,
            UomCode = l.UomId != null && uoms.ContainsKey(l.UomId.Value) ? uoms[l.UomId.Value] : null,
            Notes = l.Notes
        }).ToList();
        _db.WorkOrderBomSnapshots.AddRange(snapshots);

        // 3) 建立 Routing 快照 + 生成 Runcard 與步驟
        var ops = await _db.RoutingOperations
            .Where(x => x.RoutingHeaderId == routingHeader.Id)
            .OrderBy(x => x.Seq).ToListAsync();

        _db.WorkOrderRoutingSnapshots.AddRange(ops.Select(o => new WorkOrderRoutingSnapshot {
            WorkOrderId = wo.Id,
            Seq = o.Seq,
            WorkCenterCode = (await _db.WorkCenters.FindAsync(o.WorkCenterId))!.Code,
            OperationCode = o.OperationCode,
            StdSetupTimeMin = o.StdSetupTimeMin,
            StdRunTimeMin = o.StdRunTimeMin,
            Instruction = o.Instruction,
            CheckpointsJson = o.CheckpointsJson
        }));

        var rc = new Runcard { WorkOrderId = wo.Id, LotNo = $"{woNo}-01", Status = "Open" };
        _db.Runcards.Add(rc);
        await _db.SaveChangesAsync();

        // 4) 以 Routing 快照產生 RuncardStep
        var steps = ops.Select(o => new RuncardStep {
            RuncardId = rc.Id,
            Seq = o.Seq,
            WorkCenterCode = (await _db.WorkCenters.FindAsync(o.WorkCenterId))!.Code,
            OperationCode = o.OperationCode,
            Status = "Pending"
        }).ToList();

        _db.RuncardSteps.AddRange(steps);
        await _db.SaveChangesAsync();
        return wo.Id;
    }
}
Runcard 作業（Controller 片段）

[Route("api/runcard")]
public class RuncardController : Controller
{
    private readonly MesDbContext _db;
    public RuncardController(MesDbContext db) => _db = db;

    [HttpPost("{runcardId}/steps/{seq}/start")]
    public async Task<IActionResult> StartStep(int runcardId, int seq, [FromBody] string operatorUser)
    {
        var s = await _db.RuncardSteps.FirstAsync(x => x.RuncardId == runcardId && x.Seq == seq);
        s.Status = "Processing";
        s.StartTime = DateTime.UtcNow;
        s.OperatorUserId = operatorUser;
        await _db.SaveChangesAsync();
        return Ok();
    }

    [HttpPost("{runcardId}/steps/{seq}/complete")]
    public async Task<IActionResult> CompleteStep(int runcardId, int seq, [FromBody] CompleteStepRequest req)
    {
        var s = await _db.RuncardSteps.FirstAsync(x => x.RuncardId == runcardId && x.Seq == seq);
        s.Status = "Completed";
        s.EndTime = DateTime.UtcNow;
        s.MeasurementsJson = req.MeasurementsJson;
        s.SignOff = req.SignOff;
        await _db.SaveChangesAsync();
        return Ok();
    }
}
public record CompleteStepRequest(string? MeasurementsJson, string? SignOff);
七、工程變更（ECO/ECN）與通知
1) 流程建議（實務）
Draft：工程/製造提出變更（新增/刪除/修改 BOM 零件、Routing 工序…）。

Impact 分析：

影響哪些 已發工單？（比對 WO 的凍結版與 ECO 的 ToRevision/Effectivity）

若 Effectivity = Date，則比較 WorkOrder.CreatedAt 或 RuncardStep.StartTime；

若 Effectivity = Lot/WO，對應指定 Lot/WO；

若 Effectivity = Revision，則以版號切換。

審核/簽核（Approved）：準備通知對象（計畫/製造/檢驗/採購）。

Released：

對 未發工單：新 WO 一律使用新版本。

對 已發工單：

不追溯：維持舊版（建議做法）。

追溯：對未執行工序/未領料者，允許重新派工/替代領料；需要介面下達「re-work/補充檢驗」指示。

通知：在系統通知中心（或 Email/Slack）送達；要求「已讀/確認」。

2) 服務層（簡化：核發 ECO → 更新 Header 與明細、發通知）
public class EcoService
{
    private readonly MesDbContext _db;
    public EcoService(MesDbContext db) => _db = db;

    public async Task<int> ReleaseEcoAsync(int ecoId)
    {
        var eco = await _db.ECOs.Include(x => x.Id).FirstAsync(x => x.Id == ecoId);
        if (eco.Status != "Approved") throw new InvalidOperationException("ECO not approved.");

        // 依 ECO.Type 決定改 BOM 或 Routing：
        if (eco.Type is "BOM" or "Both")
        {
            // 新增新版 BOM Header（FromRevision -> ToRevision），把舊版複製再套 ECODetail Patch
            await CreateNextBomRevisionAsync(eco.TargetProductId, eco.FromRevision, eco.ToRevision, eco.Id);
        }
        if (eco.Type is "Routing" or "Both")
        {
            await CreateNextRoutingRevisionAsync(eco.TargetProductId, eco.FromRevision, eco.ToRevision, eco.Id);
        }

        eco.Status = "Released";
        await _db.SaveChangesAsync();

        // 通知（可擴充：查對象、寄信/SignalR）
        _db.ECNNotifications.Add(new ECNNotification {
            ECOId = eco.Id, ToUserId = "manufacturing", Channel = "UI", SentAt = DateTime.UtcNow
        });
        await _db.SaveChangesAsync();

        return eco.Id;
    }

    private async Task CreateNextBomRevisionAsync(int productId, string? fromRev, string toRev, int ecoId)
    {
        var baseHdr = await _db.BomHeaders.FirstAsync(x => x.ProductId == productId && x.Revision == fromRev);
        var newHdr = new BomHeader {
            ProductId = productId, Revision = toRev, IsReleased = true, Status = "Released",
            EffectiveFrom = DateTime.UtcNow
        };
        _db.BomHeaders.Add(newHdr);
        await _db.SaveChangesAsync();

        // 複製舊版 Lines
        var oldLines = await _db.BomLines.Where(x => x.BomHeaderId == baseHdr.Id).ToListAsync();
        var map = new Dictionary<int, int>(); // oldId -> newId

        // 先插入頂層，再遞迴（簡化：兩趟）
        foreach (var ol in oldLines.Where(x => x.ParentLineId == null).OrderBy(x => x.LineNo))
        {
            var nl = CopyLine(ol, newHdr.Id, null); _db.BomLines.Add(nl);
            await _db.SaveChangesAsync(); map[ol.Id] = nl.Id;
            await CopyChildren(oldLines, ol.Id, newHdr.Id, nl.Id, map);
        }

        // 套用 ECODetail（Add/Update/Remove）
        var details = await _db.ECODetails.Where(d => d.ECOId == ecoId && d.Target == "BOMLine").ToListAsync();
        foreach (var d in details)
        {
            switch (d.Action)
            {
                case "Add":
                    var add = JsonSerializer.Deserialize<BomLine>(d.PatchJson!);
                    add!.Id = 0; add.BomHeaderId = newHdr.Id; _db.BomLines.Add(add);
                    break;
                case "Update":
                    if (d.RefId == null) break;
                    var targetId = map.ContainsKey(d.RefId.Value) ? map[d.RefId.Value] : 0;
                    var toUpd = await _db.BomLines.FindAsync(targetId);
                    if (toUpd != null) JsonSerializer.Deserialize<BomLine>(d.PatchJson!, new JsonSerializerOptions { })!.CopyTo(toUpd);
                    break;
                case "Remove":
                    if (d.RefId == null) break;
                    var deleteId = map.ContainsKey(d.RefId.Value) ? map[d.RefId.Value] : 0;
                    var toDel = await _db.BomLines.FindAsync(deleteId);
                    if (toDel != null) _db.BomLines.Remove(toDel);
                    break;
            }
        }
        await _db.SaveChangesAsync();

        BomLine CopyLine(BomLine s, int hdrId, int? parentNewId) => new BomLine {
            BomHeaderId = hdrId, ParentLineId = parentNewId, LineNo = s.LineNo,
            ComponentProductId = s.ComponentProductId, QtyPer = s.QtyPer, UomId = s.UomId,
            ScrapFactorPct = s.ScrapFactorPct, IsOptional = s.IsOptional, RefDes = s.RefDes, Notes = s.Notes
        };

        async Task CopyChildren(List<BomLine> all, int parentOldId, int hdrId, int parentNewId, Dictionary<int,int> dict)
        {
            foreach (var ch in all.Where(x => x.ParentLineId == parentOldId).OrderBy(x => x.LineNo))
            {
                var nl = CopyLine(ch, hdrId, parentNewId); _db.BomLines.Add(nl);
                await _db.SaveChangesAsync(); dict[ch.Id] = nl.Id;
                await CopyChildren(all, ch.Id, hdrId, nl.Id, dict);
            }
        }
    }

    private async Task CreateNextRoutingRevisionAsync(int productId, string? fromRev, string toRev, int ecoId)
    {
        var baseHdr = await _db.RoutingHeaders.FirstAsync(x => x.ProductId == productId && x.Revision == fromRev);
        var newHdr = new RoutingHeader {
            ProductId = productId, Revision = toRev, IsReleased = true, Status = "Released", EffectiveFrom = DateTime.UtcNow
        };
        _db.RoutingHeaders.Add(newHdr);
        await _db.SaveChangesAsync();

        var oldOps = await _db.RoutingOperations.Where(x => x.RoutingHeaderId == baseHdr.Id).OrderBy(x => x.Seq).ToListAsync();
        foreach (var o in oldOps)
        {
            _db.RoutingOperations.Add(new RoutingOperation {
                RoutingHeaderId = newHdr.Id,
                Seq = o.Seq, WorkCenterId = o.WorkCenterId, OperationCode = o.OperationCode,
                StdSetupTimeMin = o.StdSetupTimeMin, StdRunTimeMin = o.StdRunTimeMin,
                UomId = o.UomId, Instruction = o.Instruction, CheckpointsJson = o.CheckpointsJson
            });
        }
        await _db.SaveChangesAsync();

        // 套用 ECODetail（RoutingOperation）
        var details = await _db.ECODetails.Where(d => d.ECOId == ecoId && d.Target == "RoutingOperation").ToListAsync();
        foreach (var d in details)
        {
            switch (d.Action)
            {
                case "Add":
                    var add = JsonSerializer.Deserialize<RoutingOperation>(d.PatchJson!);
                    add!.Id = 0; add.RoutingHeaderId = newHdr.Id; _db.RoutingOperations.Add(add);
                    break;
                case "Update":
                    // 這邊需靠 RefId 對應「舊版」Seq 或 Id，實務上建議以 OperationCode+Seq 做映射
                    break;
                case "Remove":
                    if (d.RefId != null) {
                        var toDel = await _db.RoutingOperations.FirstOrDefaultAsync(x => x.RoutingHeaderId == newHdr.Id && x.Id == d.RefId);
                        if (toDel != null) _db.RoutingOperations.Remove(toDel);
                    }
                    break;
            }
        }
        await _db.SaveChangesAsync();
    }
}
已發工單的影響（建議規則）

預設 不追溯：既有 WorkOrder.BomRevisionFrozen / RoutingRevisionFrozen 不變。

若 ECO 標記為「立即生效且追溯」：

對 尚未開始 的步驟：允許切換到新 Routing（需關聯替換），新增/刪除步驟。

對 已完成/進行中：標記「補強檢驗」或「重工」步驟（在 UI 讓製程工程師插入附加步驟）。

八、Razor UI：BOM Tree 與 Routing 顯示（超精簡）
BOM Tree Partial（_BomNode.cshtml）

@model BomNodeVm
<li>
  <span>@Model.LineNo - @Model.ProductName (x @Model.QtyPer)</span>
  @if (Model.Children?.Any() == true) {
    <ul>
      @foreach (var c in Model.Children) { @Html.Partial("_BomNode", c) }
    </ul>
  }
</li>
BOM 樹頁（Views/Bom/Tree.cshtml）

@model BomTreeVm
<h3>BOM @Model.ProductName  Rev @Model.Revision</h3>
<ul>
  @foreach (var n in Model.Nodes) { @Html.Partial("_BomNode", n) }
</ul>
Routing 清單（Views/Routing/Detail.cshtml）

@model RoutingVm
<h3>Routing Rev @Model.Revision</h3>
<table class="table">
<thead><tr><th>Seq</th><th>Op</th><th>WC</th><th>Setup(min)</th><th>Run(min)</th></tr></thead>
<tbody>
@foreach (var op in Model.Operations)
{
  <tr>
    <td>@op.Seq</td><td>@op.OperationCode</td><td>@op.WorkCenterId</td>
    <td>@op.StdSetupTimeMin</td><td>@op.StdRunTimeMin</td>
  </tr>
}
</tbody>
</table>
九、如何「訪問」BOM 與 Routing（API/UI）
API

POST /api/bom 建立/更新一版 BOM（附樹狀 Lines）

GET /api/bom/{productId}?rev=A&asOf=2025-09-03 取 BOM 樹

POST /api/routing 建立/更新 Routing

GET /api/routing/{productId}?rev=A 取 Routing

POST /api/workorders/release 發放工單並凍結（回傳 WO Id）

GET /api/workorders/{id}/runcard 取得 Runcard 與步驟

POST /api/runcard/{runcardId}/steps/{seq}/start|complete 執行作業

POST /api/eco/{id}/release 發布 ECO、生成新版 BOM/Routing、送通知

MVC 頁面

/Bom/Tree?productId=1&rev=A

/Routing/Detail?productId=1&rev=A

/WorkOrders/Detail/{id}（traveler）

/Eco/Detail/{id}（審核/通知）

十、落地步驟（快速路線）
建立 ASP.NET Core MVC 專案（保留 Startup.cs）

NuGet：Microsoft.EntityFrameworkCore.SqlServer、Microsoft.EntityFrameworkCore.Tools、Newtonsoft.Json（或 System.Text.Json）

ConnectionStrings：NorthwindMes 指向你現有 SQL Server（含 Northwind）

加上上述實體/DbContext/Services/Controllers/Views

套用 DDL（或 EF Migrations）

先塞 WorkCenter/Uom 範例資料 → 建立 BOM/ Routing → POST /workorders/release 發單

瀏覽 /Bom/Tree、/Routing/Detail、/WorkOrders/Detail/{id}

十一、實務小訣竅
避免 701 記憶體錯誤：

以工單為單位做 BOM/Routing 快照，後續查詢皆查快照表。

CTE 展開設 MAXRECURSION，並對原件熱點加索引。

防環：新增/更新 BOM 時，驗證 ParentLineId 不會形成循環。

ECO 生效策略：預設不追溯；若追溯，要求系統自動插入「補強檢驗/重工」步驟並通知。

通知與追蹤：ECN 需有「確認已讀」欄位，Traveler 顯示紅字提示。

權限：Draft→Released 需以角色/簽核控制。

如果你要，我可以把這套骨架整理成一個最小可跑的專案目錄（Controllers/Services/Views/SQL），再加幾組假資料與 Postman 集合，讓你直接啟動就能瀏覽 BOM/Routing 樹與發單後的 runcard。你也可以先告訴我 要以你現有的欄位命名 對齊，或我直接幫你把 BomLine／RoutingOperation 的欄位映射改成你現成的表。

