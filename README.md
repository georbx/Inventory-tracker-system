from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///inventory.db'
db = SQLAlchemy(app)

# Product model
class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    barcode = db.Column(db.String(50), unique=True, nullable=False)
    stock = db.Column(db.Integer, nullable=False, default=0)
    threshold = db.Column(db.Integer, nullable=False, default=5)

    def to_dict(self):
        return {
            'id': self.id,
            'name': self.name,
            'barcode': self.barcode,
            'stock': self.stock,
            'threshold': self.threshold
        }

# Initialize database
with app.app_context():
    db.create_all()

# Route to add/update product stock
@app.route('/stock/update', methods=['POST'])
def update_stock():
    data = request.json
    barcode = data.get('barcode')
    quantity = data.get('quantity', 0)

    product = Product.query.filter_by(barcode=barcode).first()
    if not product:
        return jsonify({'error': 'Product not found'}), 404

    product.stock += quantity
    db.session.commit()
    return jsonify({'message': 'Stock updated', 'product': product.to_dict()}), 200

# Route to add a new product
@app.route('/product/add', methods=['POST'])
def add_product():
    data = request.json
    name = data.get('name')
    barcode = data.get('barcode')
    stock = data.get('stock', 0)
    threshold = data.get('threshold', 5)

    if Product.query.filter_by(barcode=barcode).first():
        return jsonify({'error': 'Product with this barcode already exists'}), 400

    product = Product(name=name, barcode=barcode, stock=stock, threshold=threshold)
    db.session.add(product)
    db.session.commit()
    return jsonify({'message': 'Product added', 'product': product.to_dict()}), 201

# Route to simulate a product being sold (scanned)
@app.route('/product/sell', methods=['POST'])
def sell_product():
    data = request.json
    barcode = data.get('barcode')

    product = Product.query.filter_by(barcode=barcode).first()
    if not product:
        return jsonify({'error': 'Product not found'}), 404

    if product.stock <= 0:
        return jsonify({'error': 'Out of stock'}), 400

    product.stock -= 1
    db.session.commit()

    # Check low stock
    if product.stock < product.threshold:
        print(f"[⚠️ LOW STOCK ALERT] {product.name} has only {product.stock} left!")

    return jsonify({'message': 'Product sold', 'product': product.to_dict()}), 200

# Route to get product info
@app.route('/product/<barcode>', methods=['GET'])
def get_product(barcode):
    product = Product.query.filter_by(barcode=barcode).first()
    if not product:
        return jsonify({'error': 'Product not found'}), 404
    return jsonify(product.to_dict()), 200

# Route to list all products
@app.route('/products', methods=['GET'])
def list_products():
    products = Product.query.all()
    return jsonify([p.to_dict() for p in products]), 200

if __name__ == '__main__':
    app.run(debug=True)
