# MySQL: employees + sakila
git clone https://github.com/datacharmer/test_db.git && cd test_db && mysql -uroot -p < employees.sql
curl -LO https://downloads.mysql.com/docs/sakila-db.tar.gz && tar xzf sakila-db.tar.gz
mysql -uroot -p < sakila-db/sakila-schema.sql && mysql -uroot -p < sakila-db/sakila-data.sql
# PostgreSQL: pagila
git clone https://github.com/devrimgunduz/pagila.git && cd pagila
sudo -u postgres createdb pagila
sudo -u postgres psql -d pagila -f pagila-schema.sql -f pagila-data.sql
