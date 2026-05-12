# btvn3
# Nhiệm vụ 1: Thiết kế CSDL
# Nhiệm vụ 2: Cài đặt SQL (Yêu cầu viết Scripts)
## BƯỚC 1: KHỞI TẠO HỆ THỐNG BẢNG (SCHEMA)
Trước khi thực hiện các Event, chạy đoạn mã này để tạo cấu trúc dữ liệu chuẩn hóa 3NF.  
```sql
-- 1. Bảng Khách hàng [cite: 21]
CREATE TABLE KhachHang (
    MaKH INT PRIMARY KEY IDENTITY(1,1),
    HoTen NVARCHAR(100) NOT NULL,
    SDT VARCHAR(15),
    CCCD VARCHAR(20) UNIQUE
);

-- 2. Bảng Hợp đồng [cite: 22]
CREATE TABLE HopDong (
    MaHD INT PRIMARY KEY IDENTITY(1,1),
    MaKH INT FOREIGN KEY REFERENCES KhachHang(MaKH),
    TienGoc DECIMAL(18,2) NOT NULL,
    NgayVay DATETIME DEFAULT GETDATE(),
    Deadline1 DATETIME NOT NULL,
    Deadline2 DATETIME NOT NULL,
    TrangThai NVARCHAR(50) DEFAULT N'Đang vay' -- [cite: 13]
);

-- 3. Bảng Tài sản [cite: 23]
CREATE TABLE TaiSan (
    MaTS INT PRIMARY KEY IDENTITY(1,1),
    MaHD INT FOREIGN KEY REFERENCES HopDong(MaHD),
    TenTS NVARCHAR(100),
    GiaTriDinhGia DECIMAL(18,2),
    TrangThaiTS NVARCHAR(50) DEFAULT N'Đang cầm cố',
    IsSold BIT DEFAULT 0 -- [cite: 35, 49]
);

-- 4. Bảng Log biến động (Audit Log) [cite: 24, 51]
CREATE TABLE LogGiaoDich (
    MaLog INT PRIMARY KEY IDENTITY(1,1),
    MaHD INT FOREIGN KEY REFERENCES HopDong(MaHD),
    NgayGiaoDich DATETIME DEFAULT GETDATE(),
    SoTienTra DECIMAL(18,2),
    NoConLai DECIMAL(18,2),
    NguoiThuTien NVARCHAR(50)
);
```

<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/160e2d9c-55cf-48c4-b0cb-bda91199d2f6" />

## BƯỚC 2: CHI TIẾT TỪNG EVENT VÀ VÍ DỤ
- Event 1: Đăng ký hợp đồng mới (Vay tiền)   
Yêu cầu: Lưu thông tin khách hàng, tài sản, số tiền vay và thiết lập 2 mốc Deadline.
```sql
CREATE PROCEDURE sp_Event1_DangKyHopDong
    @MaKH INT,
    @TienGoc DECIMAL(18,2),
    @Deadline1 DATETIME,
    @Deadline2 DATETIME,
    @TenTS NVARCHAR(100),
    @GiaTriTS DECIMAL(18,2)
AS
BEGIN
    INSERT INTO HopDong (MaKH, TienGoc, Deadline1, Deadline2)
    VALUES (@MaKH, @TienGoc, @Deadline1, @Deadline2);

    DECLARE @NewHD INT = SCOPE_IDENTITY();
    INSERT INTO TaiSan (MaHD, TenTS, GiaTriDinhGia)
    VALUES (@NewHD, @TenTS, @GiaTriTS);
END;
```

<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/02bce737-8b83-44ec-b04f-3247a6d57d85" />

Ví dụ: Đăng ký cho ông Nguyễn Văn A vay 10.000.000đ, thế chấp xe máy.

<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/4a6b6c27-accf-4e30-b4a5-93cfe2a18db9" />

- Event 2: Tính toán công nợ thời gian thực   
```sql
Yêu cầu: Tính lãi đơn (5.000đ/1tr/ngày) trước Deadline 1 và lãi kép sau Deadline 1.
CREATE FUNCTION fn_Event2_CalcMoney (@MaHD INT, @TargetDate DATETIME)
RETURNS DECIMAL(18,2)
AS
BEGIN
    DECLARE @Goc DECIMAL(18,2), @NgayVay DATETIME, @D1 DATETIME;
    SELECT @Goc = TienGoc, @NgayVay = NgayVay, @D1 = Deadline1 FROM HopDong WHERE MaHD = @MaHD;

    DECLARE @LaiSuatNgay FLOAT = 5000.0 / 1000000.0;
    -- Tính lãi đơn [cite: 11]
    DECLARE @NgayDon INT = DATEDIFF(DAY, @NgayVay, CASE WHEN @TargetDate < @D1 THEN @TargetDate ELSE @D1 END);
    DECLARE @TongSauLaiDon DECIMAL(18,2) = @Goc + (@Goc * @LaiSuatNgay * @NgayDon);

    IF (@TargetDate <= @D1) RETURN @TongSauLaiDon;
    -- Tính lãi kép từ mốc D1 [cite: 12]
    DECLARE @NgayKep INT = DATEDIFF(DAY, @D1, @TargetDate);
    RETURN @TongSauLaiDon * POWER(1 + @LaiSuatNgay, @NgayKep);
END;
```
<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/29cdf1f1-08b5-431f-8861-ef4a4d63c871" />


Ví dụ: Tính xem đến ngày 20/05 ông A phải trả bao nhiêu.

<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/921b362d-ec52-46b7-a599-68c78147dcd1" />

- Event 3: Xử lý trả nợ và hoàn trả tài sản   
Yêu cầu: Trừ tiền trả vào hệ thống, ghi Log, kiểm tra điều kiện trả đồ (Giá trị TS còn lại >= Dư nợ).  
```sql
CREATE PROCEDURE sp_Event3_XuLyTraNo
    @MaHD INT,
    @SoTienTra DECIMAL(18,2)
AS
BEGIN
    -- Kiểm tra cờ thanh lý [cite: 35]
    IF EXISTS (SELECT 1 FROM TaiSan WHERE MaHD = @MaHD AND IsSold = 1)
    BEGIN
        PRINT N'Tài sản đã bán, không thu tiền'; RETURN;
    END

    DECLARE @TongNo DECIMAL(18,2) = dbo.fn_Event2_CalcMoney(@MaHD, GETDATE());
    DECLARE @DuNo DECIMAL(18,2) = @TongNo - @SoTienTra;

    -- Ghi Log giao dịch [cite: 51]
    INSERT INTO LogGiaoDich (MaHD, SoTienTra, NoConLai) VALUES (@MaHD, @SoTienTra, @DuNo);

    -- Cập nhật trạng thái hợp đồng [cite: 37, 38]
    UPDATE HopDong SET TrangThai = CASE WHEN @DuNo <= 0 THEN N'Đã thanh toán đủ' ELSE N'Đang trả góp' END WHERE MaHD = @MaHD;

    -- Gợi ý trả đồ dựa trên điều kiện giá trị [cite: 39, 40]
    SELECT MaTS, TenTS FROM TaiSan WHERE MaHD = @MaHD AND GiaTriDinhGia >= @DuNo;
END;

```


<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/84c9f8c0-afc0-4b4b-a63b-438c590b83da" />


Ví dụ: Ông A đến trả trước 5.000.000đ.
<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/9f2be4c7-2f8d-46d1-955c-cfb1b39afa36" />

- Event 4: Truy vấn danh sách nợ xấu   

Yêu cầu: Xuất danh sách khách quá Deadline 1, tính số ngày quá hạn và tiền phải trả sau 1 tháng.
```sql
CREATE PROCEDURE sp_Event4_QueryBadDebt
AS
BEGIN
    SELECT 
        KH.HoTen, KH.SDT, HD.TienGoc,
        DATEDIFF(DAY, HD.Deadline1, GETDATE()) AS SoNgayQuaHan,
        dbo.fn_Event2_CalcMoney(HD.MaHD, GETDATE()) AS NoHienTai,
        dbo.fn_Event2_CalcMoney(HD.MaHD, DATEADD(MONTH, 1, GETDATE())) AS NoDuKienSau1Thang
    FROM HopDong HD
    JOIN KhachHang KH ON HD.MaKH = KH.MaKH
    WHERE GETDATE() > HD.Deadline1 AND HD.TrangThai != N'Đã thanh toán đủ';
END;
```

<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/7b6d0587-e7ff-4496-894b-a4b33f8a3413" />

Ví dụ: Xem danh sách nợ xấu hiện tại.
```sql
EXEC sp_Event4_QueryBadDebt;
```
<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/6aa9c6fc-5e1d-4f95-8983-469dda61cfad" />

- Event 5: Quản lý thanh lý tài sản (Trigger)   
Yêu cầu: Tự động chuyển trạng thái "Quá hạn" sau Deadline 1 và "Sẵn sàng thanh lý" sau Deadline 2.

```sql
-- Trigger 1: Tự động chuyển sang "Quá hạn (nợ xấu)" sau Deadline 1 [cite: 46]
CREATE TRIGGER trg_Event5_CheckDeadline1
ON HopDong
AFTER UPDATE, INSERT
AS
BEGIN
    UPDATE HopDong
    SET TrangThai = N'Quá hạn (nợ xấu)'
    FROM HopDong H
    JOIN inserted i ON H.MaHD = i.MaHD
    WHERE i.TrangThai = N'Đang vay' 
      AND GETDATE() > i.Deadline1; -- So sánh với ngày hiện tại [cite: 46]
END;
GO

-- Trigger 2: Tự động chuyển tài sản sang "Sẵn sàng thanh lý" sau Deadline 2 [cite: 47]
CREATE TRIGGER trg_Event5_CheckDeadline2
ON HopDong
AFTER UPDATE
AS
BEGIN
    UPDATE TaiSan
    SET TrangThaiTS = N'Sẵn sàng thanh lý'
    FROM TaiSan T
    JOIN inserted i ON T.MaHD = i.MaHD
    WHERE i.TrangThai = N'Quá hạn (nợ xấu)' 
      AND GETDATE() > i.Deadline2; -- Vượt quá mốc thanh lý [cite: 47]
END;
GO
```

<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/ce3ee646-cfae-443e-a459-7f2609d1227f" />

<img width="960" height="600" alt="image" src="https://github.com/user-attachments/assets/6cf9a0d6-dfc6-4840-8b84-b1dfe2b5b77b" />


