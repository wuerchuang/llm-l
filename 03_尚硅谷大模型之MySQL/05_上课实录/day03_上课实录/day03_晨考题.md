1. 在t_employee表中查询和 白露，谢吉娜同一部门的员工姓名和电话
2. 假设有一个employees表，包含employee_id（员工编号）、first_name（名字）、last_name（姓氏）、salary（工资）、department_id（部门编号）字段。编写 SQL 语句，将部门编号为10的员工工资提高 10%。
3. 假设有一个orders表，包含order_id（订单编号）、customer_id（客户编号）、order_date（订单日期）、order_amount（订单金额）字段。编写 SQL 语句，计算每个客户的订单总金额，并按总金额从高到低显示客户编号和总金额。
4. 假设有两个表customers（包含customer_id、customer_name）和orders（包含order_id、customer_id、order_date）。编写 SQL 语句，使用内连接查询出每个客户的姓名及其对应的订单日期。
5. 假设有products表（包含product_id、product_name、category_id、price）和categories表（包含category_id、category_name）。编写 SQL 语句，使用左外连接查询所有产品类别及其对应的产品名称（如果该类别有产品），若某类别没有产品，产品名称显示为 NULL。