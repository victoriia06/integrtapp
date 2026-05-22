-- =====================================================
-- ПОЛНЫЙ SQL-СКРИПТ ДЛЯ ИНТЕРНЕТ-МАГАЗИНА / СКЛАДА
-- Выполнить в pgAdmin (база данных Warehouse)
-- =====================================================

-- =====================================================
-- 1. УДАЛЕНИЕ СТАРЫХ ТАБЛИЦ (если нужно пересоздать с нуля)
-- =====================================================
-- ВНИМАНИЕ: эти команды удалят все данные!
-- Раскомментируйте, если нужно пересоздать всё заново

/*
DROP TABLE IF EXISTS OrderInfo CASCADE;
DROP TABLE IF EXISTS ClientOrder CASCADE;
DROP TABLE IF EXISTS ReceiptInfo CASCADE;
DROP TABLE IF EXISTS Receipt CASCADE;
DROP TABLE IF EXISTS Stock CASCADE;
DROP TABLE IF EXISTS Product CASCADE;
DROP TABLE IF EXISTS Client CASCADE;
DROP TABLE IF EXISTS Supplier CASCADE;
*/

-- =====================================================
-- 2. СОЗДАНИЕ ТАБЛИЦ
-- =====================================================

-- Товары
CREATE TABLE IF NOT EXISTS Product (
    ID SERIAL PRIMARY KEY,
    Name VARCHAR(100) NOT NULL,
    Ed VARCHAR(20) NOT NULL
);

-- Поставщики
CREATE TABLE IF NOT EXISTS Supplier (
    ID SERIAL PRIMARY KEY,
    Name VARCHAR(100) NOT NULL,
    Address VARCHAR(200),
    Phone VARCHAR(20)
);

-- Клиенты
CREATE TABLE IF NOT EXISTS Client (
    ID SERIAL PRIMARY KEY,
    Name VARCHAR(100) NOT NULL,
    Address VARCHAR(200),
    Phone VARCHAR(20)
);

-- Остатки на складе
CREATE TABLE IF NOT EXISTS Stock (
    ProductID INT PRIMARY KEY REFERENCES Product(ID) ON DELETE CASCADE,
    Quantity NUMERIC(12,3) DEFAULT 0
);

-- Приходная накладная
CREATE TABLE IF NOT EXISTS Receipt (
    ID SERIAL PRIMARY KEY,
    Date DATE NOT NULL,
    SupplierID INT REFERENCES Supplier(ID) ON DELETE RESTRICT,
    TotalSum NUMERIC(15,2) DEFAULT 0
);

-- Содержимое приходной накладной
CREATE TABLE IF NOT EXISTS ReceiptInfo (
    ID SERIAL PRIMARY KEY,
    ReceiptID INT REFERENCES Receipt(ID) ON DELETE CASCADE,
    ProductID INT REFERENCES Product(ID) ON DELETE RESTRICT,
    Quantity NUMERIC(12,3) NOT NULL,
    Price NUMERIC(12,2) NOT NULL
);

-- Заказы клиентов
CREATE TABLE IF NOT EXISTS ClientOrder (
    ID SERIAL PRIMARY KEY,
    Number VARCHAR(20) NOT NULL,
    OrderDate DATE NOT NULL,
    ClientID INT REFERENCES Client(ID) ON DELETE RESTRICT,
    TotalSum NUMERIC(15,2) DEFAULT 0,
    Status VARCHAR(20) DEFAULT 'Новый'
);

-- Содержимое заказа
CREATE TABLE IF NOT EXISTS OrderInfo (
    ID SERIAL PRIMARY KEY,
    OrderID INT REFERENCES ClientOrder(ID) ON DELETE CASCADE,
    ProductID INT REFERENCES Product(ID) ON DELETE RESTRICT,
    Quantity NUMERIC(12,3) NOT NULL,
    Price NUMERIC(12,2) NOT NULL
);

-- =====================================================
-- 3. ТРИГГЕРНЫЕ ФУНКЦИИ
-- =====================================================

-- Триггер 1: При добавлении товара в приходную накладную
CREATE OR REPLACE FUNCTION update_receipt_total_insert()
RETURNS TRIGGER AS $$
BEGIN
    -- Обновляем сумму накладной
    UPDATE Receipt 
    SET TotalSum = TotalSum + NEW.Quantity * NEW.Price
    WHERE ID = NEW.ReceiptID;
    
    -- Обновляем остатки на складе (увеличиваем)
    INSERT INTO Stock (ProductID, Quantity) 
    VALUES (NEW.ProductID, NEW.Quantity)
    ON CONFLICT (ProductID) 
    DO UPDATE SET Quantity = Stock.Quantity + NEW.Quantity;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Триггер 2: При удалении товара из приходной накладной
CREATE OR REPLACE FUNCTION update_receipt_total_delete()
RETURNS TRIGGER AS $$
BEGIN
    -- Обновляем сумму накладной
    UPDATE Receipt 
    SET TotalSum = TotalSum - OLD.Quantity * OLD.Price
    WHERE ID = OLD.ReceiptID;
    
    -- Обновляем остатки на складе (уменьшаем)
    UPDATE Stock 
    SET Quantity = Quantity - OLD.Quantity
    WHERE ProductID = OLD.ProductID;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Триггер 3: При добавлении товара в заказ (с проверкой остатков)
CREATE OR REPLACE FUNCTION check_stock_and_update_order()
RETURNS TRIGGER AS $$
DECLARE
    stock_qty NUMERIC;
BEGIN
    -- Проверяем остаток на складе
    SELECT COALESCE(Quantity, 0) INTO stock_qty FROM Stock WHERE ProductID = NEW.ProductID;
    
    IF stock_qty < NEW.Quantity THEN
        RAISE EXCEPTION 'Недостаточно товара "%" на складе. Доступно: %, запрошено: %',
            (SELECT Name FROM Product WHERE ID = NEW.ProductID),
            stock_qty, NEW.Quantity;
    END IF;
    
    -- Обновляем сумму заказа
    UPDATE ClientOrder 
    SET TotalSum = TotalSum + NEW.Quantity * NEW.Price
    WHERE ID = NEW.OrderID;
    
    -- Резервируем товар (уменьшаем склад)
    UPDATE Stock 
    SET Quantity = Quantity - NEW.Quantity
    WHERE ProductID = NEW.ProductID;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Триггер 4: При изменении статуса заказа на "Отменен" (возврат товаров)
CREATE OR REPLACE FUNCTION cancel_order_return_stock()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.Status != 'Отменен' AND NEW.Status = 'Отменен' THEN
        UPDATE Stock s
        SET Quantity = s.Quantity + oi.Quantity
        FROM OrderInfo oi
        WHERE oi.OrderID = NEW.ID AND s.ProductID = oi.ProductID;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- =====================================================
-- 4. СОЗДАНИЕ ТРИГГЕРОВ
-- =====================================================

-- Триггер на добавление в накладную
DROP TRIGGER IF EXISTS ins_receipt_info ON ReceiptInfo;
CREATE TRIGGER ins_receipt_info
AFTER INSERT ON ReceiptInfo
FOR EACH ROW
EXECUTE FUNCTION update_receipt_total_insert();

-- Триггер на удаление из накладной
DROP TRIGGER IF EXISTS del_receipt_info ON ReceiptInfo;
CREATE TRIGGER del_receipt_info
AFTER DELETE ON ReceiptInfo
FOR EACH ROW
EXECUTE FUNCTION update_receipt_total_delete();

-- Триггер на добавление в заказ (с проверкой остатков)
DROP TRIGGER IF EXISTS ins_order_info ON OrderInfo;
CREATE TRIGGER ins_order_info
BEFORE INSERT ON OrderInfo
FOR EACH ROW
EXECUTE FUNCTION check_stock_and_update_order();

-- Триггер на отмену заказа
DROP TRIGGER IF EXISTS cancel_order ON ClientOrder;
CREATE TRIGGER cancel_order
BEFORE UPDATE OF Status ON ClientOrder
FOR EACH ROW
EXECUTE FUNCTION cancel_order_return_stock();

-- =====================================================
-- 5. ТЕСТОВЫЕ ДАННЫЕ
-- =====================================================

-- Товары
INSERT INTO Product (ID, Name, Ed) VALUES 
(1, 'Ноутбук', 'шт'),
(2, 'Мышь', 'шт'),
(3, 'Клавиатура', 'шт'),
(4, 'Монитор', 'шт'),
(5, 'USB-кабель', 'шт')
ON CONFLICT (ID) DO NOTHING;

-- Поставщики
INSERT INTO Supplier (ID, Name, Address, Phone) VALUES
(1, 'ООО "Компьютеры+"', 'г. Москва, ул. Ленина, 1', '+7(495)123-45-67'),
(2, 'ИП Иванов', 'г. Санкт-Петербург, Невский пр., 2', '+7(812)987-65-43')
ON CONFLICT (ID) DO NOTHING;

-- Клиенты
INSERT INTO Client (ID, Name, Address, Phone) VALUES
(1, 'ООО "Ромашка"', 'г. Москва, ул. Пушкина, 10', '+7(495)111-22-33'),
(2, 'ИП Петров', 'г. Казань, ул. Баумана, 5', '+7(843)555-66-77')
ON CONFLICT (ID) DO NOTHING;

-- Начальные остатки на складе
INSERT INTO Stock (ProductID, Quantity) VALUES
(1, 10),
(2, 50),
(3, 30),
(4, 5),
(5, 100)
ON CONFLICT (ProductID) DO NOTHING;

-- =====================================================
-- 6. ПРОВЕРКА
-- =====================================================

-- Посмотреть все триггеры
SELECT 
    tgname AS trigger_name,
    tgrelid::regclass AS table_name,
    tgfoid::regproc AS function_name
FROM pg_trigger
WHERE tgrelid::regclass::text IN ('orderinfo', 'receiptinfo', 'clientorder')
ORDER BY table_name;

-- Посмотреть таблицы
SELECT COUNT(*) as product_count FROM Product;
SELECT COUNT(*) as supplier_count FROM Supplier;
SELECT COUNT(*) as client_count FROM Client;
SELECT COUNT(*) as stock_count FROM Stock;
