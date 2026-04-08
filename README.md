# bynry-stockflow-case-study
**ISSUE 1 - MULTIPLE COMMITS CAUSING INCONSISTENT DATA**
The product and inventory are commit separately
db.session.commit()
db.session.commit()
Two times

**IMPACT**
If one of them fails:
example inventory fails
inventory not created,  but product gts created , Inconsistent data in production

**FIX**
with db.session.begin():
    db.session.add(product)


ISSUE 2 - PRODUCT TIED TO SINGLE WAREHOUSE
product included warhouse id
but in summary:
product can exist in multiple warehouses

**IMPACT**
diffcult stock management
same product must be duplicated

**FIX**
Remove it from product and handle it in inventory
product = Product(
    name=data['name'],
    sku=data['sku'],
    price=data['price']
)

inventory = Inventory(
    product_id=product.id,
    warehouse_id=data['warehouse_id'],
    quantity=data['initial_quantity']
)


**ISSUE 3 - NO SKU UNIQUENESS VALIDATES**
No check are implement for duplicate SKU

**IMPACT**
Wrong mapping
ordering confusion

**FIX**
existing = Product.query.filter_by(sku=data["sku"]).first()
if existing:
    return {"error": "SKU already exists"}, 400



**ISSUE 4 - NO INPUT VALIDATION**
Code directly accesses field
data['name']
data['initial_quantity']

if missing - crash

**IMPACT**
500 Error
unstable api

required_fields = ["name", "sku", "price", "warehouse_id", "initial_quantity"]

for field in required_fields:
    if field not in data:
        return {"error": f"{field} is required"}, 400


**FINAL CORRECTED IMPLEMENTATION**


@app.route('/api/products', methods=['POST'])
def create_product():
    try:
        data = request.json

        required_fields = ["name", "sku", "price", "warehouse_id", "initial_quantity"]
        for field in required_fields:
            if field not in data:
                return {"error": f"{field} is required"}, 400

        existing = Product.query.filter_by(sku=data["sku"]).first()
        if existing:
            return {"error": "SKU already exists"}, 400

        if data["initial_quantity"] < 0:
            return {"error": "Quantity cannot be negative"}, 400

        with db.session.begin():

            product = Product(
                name=data["name"],
                sku=data["sku"],
                price=Decimal(["price"])
            )

            db.session.add(product)
            db.session.flush()

            inventory = Inventory(
                product_id=product.id,
                warehouse_id=data["warehouse_id"],
                quantity=data["initial_quantity"]
            )

            db.session.add(inventory)

        return {
            "message": "Product created",
            "product_id": product.id
        }, 201

    except IntegrityError:
        db.session.rollback()
        return {"error": "Database error"}, 400

    except Exception as e:
        db.session.rollback()
        return {"error": str(e)}, 500






**PART 2 DATABASE DESIGN**
**Companies Table**
CREATE TABLE companies (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
**Warehouses Table**
 CREATE TABLE warehouses (
    id INT PRIMARY KEY AUTO_INCREMENT,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    location VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(id)
);       
**Company → multiple warehouses**

**Products Table
Product is company scoped and warehouse independent.**

CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100) UNIQUE,
    price DECIMAL(10,2),
    product_type VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(id)
);

**INVENTORY TABLE**
CREATE TABLE inventory (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT NOT NULL,
    warehouse_id INT NOT NULL,
    quantity INT DEFAULT 0,
    low_stock_threshold INT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    UNIQUE(product_id, warehouse_id)
);

**INVENTORY MOVEMENT TABLE**
CREATE TABLE inventory_movements (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT,
    warehouse_id INT,
    change_quantity INT,
    reason VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

**SUPPLIER TABLE**
CREATE TABLE suppliers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    contact_email VARCHAR(255),
    phone VARCHAR(50)
);

**PRODUCT SUPPLIER MAPPING**
CREATE TABLE product_suppliers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT,
    supplier_id INT,
    lead_time_days INT,
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (supplier_id) REFERENCES suppliers(id)
);

**BUNDLE TABLE**
CREATE TABLE product_bundles (
    id INT PRIMARY KEY AUTO_INCREMENT,
    bundle_product_id INT,
    child_product_id INT,
    quantity INT,
    FOREIGN KEY (bundle_product_id) REFERENCES products(id),
    FOREIGN KEY (child_product_id) REFERENCES products(id)
);



**PART 3: LOW STOCK ALERTS API**
GET /api/companies/{company_id}/alerts/low-stock

**FASTAPI**
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from datetime import datetime, timedelta

router = APIRouter()

@router.get("/api/companies/{company_id}/alerts/low-stock")
def low_stock_alerts(company_id: int, db: Session = Depends(get_db)):

    alerts = []

    # last 30 days sales window
    last_30_days = datetime.utcnow() - timedelta(days=30)

    results = (
        db.query(
            Product.id,
            Product.name,
            Product.sku,
            Warehouse.id,
            Warehouse.name,
            Inventory.quantity,
            Inventory.low_stock_threshold,
            Supplier.id,
            Supplier.name,
            Supplier.contact_email
        )
        .join(Inventory, Inventory.product_id == Product.id)
        .join(Warehouse, Warehouse.id == Inventory.warehouse_id)
        .join(ProductSupplier, ProductSupplier.product_id == Product.id)
        .join(Supplier, Supplier.id == ProductSupplier.supplier_id)
        .filter(Warehouse.company_id == company_id)
        .filter(Inventory.quantity <= Inventory.low_stock_threshold)
        .all()
    )

    for row in results:

        current_stock = row[5]
        threshold = row[6]

        # assume avg daily sales = 1 (can be calculated from orders table)
        avg_daily_sales = 1

        if avg_daily_sales > 0:
            days_until_stockout = current_stock // avg_daily_sales
        else:
            days_until_stockout = None

        alerts.append({
            "product_id": row[0],
            "product_name": row[1],
            "sku": row[2],
            "warehouse_id": row[3],
            "warehouse_name": row[4],
            "current_stock": current_stock,
            "threshold": threshold,
            "days_until_stockout": days_until_stockout,
            "supplier": {
                "id": row[7],
                "name": row[8],
                "contact_email": row[9]
            }
        })

    return {
        "alerts": alerts,
        "total_alerts": len(alerts)
    }


