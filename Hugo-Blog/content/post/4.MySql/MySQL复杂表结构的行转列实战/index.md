---
title: MySQL复杂表结构的行转列实战
description: 经常会遇到需要将复杂表结构的数据进行行转列（Pivot）的需求，尤其是在报表统计、数据透视等场景下。我们可以通过临时表、动态SQL等方式把复杂问题简单化，实现灵活的行转列
date: 2025-08-31
slug: MySql/2025-08-31
image: 3.jpg
categories:
    - MySQL
    - 行转列
    - 数据库
tags:
    - MySQL
    - 数据库
---

# MySQL复杂表结构的行转列实战

经常会遇到需要将复杂表结构的数据进行行转列（Pivot）的需求，尤其是在报表统计、数据透视等场景下。我们可以通过临时表、动态SQL等方式把复杂问题简单化，实现灵活的行转列。

## 1. 问题背景

假设我们有一个项目工时表 `project_time`，记录了每个员工在不同项目、不同日期的工时。表结构如下：

- `project_time`：项目工时明细（包含项目ID、员工ID、日期、工时等）
- `project`：项目信息
- `department`：部门信息
- `employee`：员工信息

我们的目标是：
- 以项目为主轴，统计每个员工在每一天的工时（即员工为列，日期为动态列，工时为值）。

## 2. 数据准备与临时表构建

首先，为了简化后续操作，将多表关联后的结果存入临时表：

```sql
-- 删除临时表（如果存在）
DROP TEMPORARY TABLE IF EXISTS temp_project_time;

-- 创建临时表
CREATE TEMPORARY TABLE temp_project_time AS 
SELECT 
    pt.project_id,
    p.organization_id,
    d.`name` AS department_name,
    p.`code` AS project_code,
    p.`name` AS project_name,
    p.start_date,
    p.end_date,
    e.`name` AS employee_name,
    pt.work_date,
    pt.work_hour 
FROM project_time AS pt 
LEFT JOIN project AS p ON pt.project_id = p.id 
LEFT JOIN department d ON p.department_id = d.id 
LEFT JOIN employee e ON pt.employee_id = e.id 
WHERE 1=1
    AND (p_start_date IS NULL OR pt.work_date >= p_start_date)
    AND (p_end_date IS NULL OR pt.work_date <= p_end_date)
    AND (p_employee_id IS NULL OR p_employee_id = 0 OR pt.employee_id = p_employee_id)
    AND (p_project_id IS NULL OR p_project_id = 0 OR pt.project_id = p_project_id)
    AND (p_department_id IS NULL OR p_department_id = 0 OR p.department_id = p_department_id)
    AND (p_search_key IS NULL OR p_search_key = '' OR 
      p.`code` LIKE CONCAT('%', p_search_key, '%') COLLATE utf8_general_ci OR 
        p.`name` LIKE CONCAT('%', p_search_key, '%') COLLATE UTF8_GENERAL_CI )
GROUP BY 
    pt.project_id, 
    p.organization_id,
    d.`name`,
    p.`code`,
    p.`name`,
    p.start_date,
    p.end_date,
    e.`name`,
    pt.work_date,
    pt.work_hour 
ORDER BY pt.work_date DESC, pt.project_id DESC;
```

## 3. 静态行转列（CASE WHEN）

如果日期是固定，可以直接用条件聚合：

```sql
SELECT
  project_id,
  project_name,
  work_date,
  SUM(CASE WHEN work_date = '2025-01-01' THEN work_hour ELSE 0 END) AS '2025-01-01',
  SUM(CASE WHEN work_date = '2025-01-02' THEN work_hour ELSE 0 END) AS '2025-01-02',
  SUM(CASE WHEN work_date = '2025-01-03' THEN work_hour ELSE 0 END) AS '2025-01-03'
FROM temp_project_time
GROUP BY project_id, project_name, work_date
ORDER BY work_date DESC, project_id DESC;
```

## 4. 动态行转列（动态SQL）

实际业务中，员工数量和日期往往不固定，需要动态生成列。可通过MySQL动态SQL实现：

```sql
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_sql TEXT DEFAULT '';
    -- 删除临时表（如果存在）
    DROP TEMPORARY TABLE IF EXISTS temp_project_time;
    -- 创建临时表
    CREATE TEMPORARY TABLE temp_project_time AS ... -- 见上文
    -- 生成动态日期列
    SET @EE = '';
    SELECT 
        GROUP_CONCAT(
            CONCAT('sum(if(work_date=\'', work_date, '\',work_hour,0)) as `', work_date, '`')
            ORDER BY work_date
            SEPARATOR ', '
        ) INTO @EE
    FROM (
        SELECT DISTINCT DATE_FORMAT(work_date, '%Y-%m-%d') AS work_date 
        FROM temp_project_time
        ORDER BY work_date
    ) AS date_list;
    IF @EE IS NULL OR @EE = '' THEN
        SET @EE = '0 AS no_data';
    END IF;
    -- 构建最终查询SQL
    SET @QQ = CONCAT(
        'SELECT ',
        '    project_id, ',
        '    organization_id, ',
        '    department_name, ',
        '    project_code, ',
        '    project_name, ',
        '    start_date, ',
        '    end_date, ',
        '    employee_name, ',
        @EE, ', ',
        '    sum(work_hour) AS TOTAL ',
        'FROM temp_project_time ',
        'GROUP BY ',
        '    project_id, ',
        '    organization_id, ',
        '    department_name, ',
        '    project_code, ',
        '    project_name, ',
        '    start_date, ',
        '    end_date, ',
        '    employee_name ',
        'ORDER BY employee_name desc,project_id desc'
    );
    PREPARE stmt FROM @QQ;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    DROP TEMPORARY TABLE IF EXISTS temp_project_time;
END
```

### 关键点说明
- 通过 `GROUP_CONCAT` 动态拼接所有日期列，自动适配不同时间段。
- 动态SQL拼接后用 `PREPARE` 和 `EXECUTE` 执行。
- 支持任意员工、任意日期的灵活行转列。

## 5. 总结

通过临时表和动态SQL，可以优雅地解决复杂表结构下的行转列需求。实际应用中可根据业务灵活调整分组和列生成逻辑，极大提升报表开发效率。

然后可以考虑结合存储过程进一步封装。