<?php

error_reporting(0);

require "../../lib/config.php";

session_start(); 
$p_id = $_SESSION['userId'];

$action = $_REQUEST['action'];

//В этом файле особо описывать нечего, ключевые моменты:
//1. Для избежания инъекций используется bind параметров к запросам и процедурам
//2. Все данные конвертируется в JSON путем сериализации объекта PHP
//3. Id текущего пользователя хранится в сессии, и сохраняется при входе в систему ($p_id)
//4. Действие GET_FILLING выводит обычную html таблицу в связи с большим объемом данных и слабыми целевыми машинами
//5. Большая часть логики приложения выделена в СУБД, PHP выступает в качестве сервиса получения JSON данных из базы.
//Не считаю целесообразным для таких целей использовать в PHP ООП подход, т.к. каждое обращение к серверу является 
//атомарным, и разворачивание большого количества объектов на один запрос пустая трата ресурсов сервера.
//ООП в PHP целесообразно использовать в случае частого использования кода в разных модулях, например: подключение к 
//базе, работа с какими-то форматами данных, система маршрутов (routes) и тд.
switch($action){
	case 'GET_PHARMACIES':
		$obj = array("units" => array());
		
		$employeeId = $_REQUEST['employeeId'];
		$employeeId = $employeeId > 0 ? $employeeId : $p_id;
		
		$result = $db_connection->prepare("call filter_userPoints(?);");
		$result->bind_param('i', $employeeId);
		$result->execute();

		$result = $db_connection->query("select p.id, p.name, p.address, p.town, if(pr.id is null, 0, 1) as checked, pr.approved from 
									t_filteredPoints t 
									inner join points p on p.id=t.id 
									left outer join projects_registeredPoints pr on pr.pointsId=t.id limit 200;");
		$obj['count'] = $result->num_rows;	
		while ($row = $result->fetch_array()) 
		{
			array_push($obj['units'], array(id => $row['id'], name => $row['name']));
			$obj['units'][count($obj['units'])-1]['address'] = $row['address'];
			$obj['units'][count($obj['units'])-1]['town'] = $row['town'];
			$obj['units'][count($obj['units'])-1]['checked'] = $row['checked'];
			$obj['units'][count($obj['units'])-1]['approved'] = $row['approved'];
		}
		$result->free();
		
		break;

	case 'GET_ACTIVE_PHARMACIES':
		$p_projectId = $_REQUEST['projectId'];
		$p_stageId = $_REQUEST['stageId'];
		$employeeId = $_REQUEST['employeeId'];
		$employeeId = $employeeId > 0 ? $employeeId : $p_id;
		
		$obj = array("units" => array());
		
		$result = $db_connection->prepare("call filter_userPoints(?);");
		$result->bind_param('i', $employeeId);
		$result->execute();

		$result = $db_connection->prepare("select p.id, p.name, p.address, p.town from 
									t_filteredPoints t 
									inner join points p on p.id=t.id 
									inner join projects_registeredPoints pr on pr.approved = 1 and pr.pointsId=t.id and pr.projectsId = ? and (pr.projectStagesId = ? OR pr.projectStagesId is NULL) order by p.town, p.name limit 200;");
		$result->bind_param('ii', $p_projectId, $p_stageId);
		$result->execute();
		
		$result->bind_result($id, $name, $address, $town);

		$count = 0;
		while ($result->fetch()) {
			array_push($obj['units'], array(id => $id, name => $name));
			$obj['units'][count($obj['units'])-1]['address'] = $address;
			$obj['units'][count($obj['units'])-1]['town'] = $town;		
			$count++;
		}
		$obj['count'] = $count;
		
		$result->close();
		
		break;

	case 'REGISTER_PHARMACY':
		$pharm = $_REQUEST['pharmId'];
		$project = $_REQUEST['projectId'];
		$registered = $_REQUEST['registered'];
		if($registered==1)
		{
			$sql = "insert into projects_registeredPoints (projectsId, pointsId) values (?, ?);";
		} else {
			$sql = "delete from projects_registeredPoints where projectsId=? and pointsId=?;";
		}
		$result = $db_connection->prepare($sql);
		$result->bind_param('ii', $project, $pharm);
		$result->execute();
		$result->close();

		$obj = new stdClass();
		$obj->result = true;
		
		break;

	case 'GET_FORMDATA':
		$projectId = $_REQUEST['projectId'];
		$stageId = $_REQUEST['stageId'];
		$pharmId = $_REQUEST['pharmId'];
		
		$sql = "call data_getFormRows (?, ?, ?);";
		$result = $db_connection->prepare($sql);
		$result->bind_param('iii', $projectId, $stageId, $pharmId);
		$result->execute();
		
		$result->bind_result($id, $name, $field, $value);

		$obj = new stdClass();
		$prev = 0;
		$element = array();
		$obj->rows = array();
		
		while ($result->fetch()) {
			if (count($obj->rows) == 0 || $obj->rows[count($obj->rows)-1]->id != $id){
				array_push($obj->rows, new stdClass()); 
				$obj->rows[count($obj->rows)-1]->id = $id;
				$obj->rows[count($obj->rows)-1]->name = $name;	
			}
			$obj->rows[count($obj->rows)-1]->{$field} = $value;
		}
		
		$result->close();
		
		break;

	case 'SET_FORMDATA':
		$projectId = $_REQUEST['projectId'];
		$stageId = $_REQUEST['stageId'];
		$pharmId = $_REQUEST['pharmId'];
		$values = $_REQUEST['values'];
		$KpiId = $_REQUEST['id'];
		$obj = new stdClass();
		
		$db_connection->query("START TRANSACTION;");
		
		for ($i = 0; $i < count($KpiId); $i++){
			foreach ($values[$i] as $key => $value){
				$sql = "select data_setNumericField(?, ?, ?, ?, ?, ?, ?);";
				$kpi2stage = str_replace('f_', '', $key);			
				$result = $db_connection->prepare($sql);
				$result->bind_param('iiiiiii', $p_id, $projectId, $stageId, $pharmId, $kpi2stage, $KpiId[$i], $value);
				$result->execute();
				$result->bind_result($headId);
				$result->fetch();

				$result->close();

				$obj->result = (isset($headId) && $headId > 0);
			}
		}
		
		if ($obj->result == true){
			$db_connection->query("COMMIT;");
		}else{
			$db_connection->query("ROLLBACK;");
		}
		
		$force = 0;
		$result = $db_connection->prepare("call formula_recalcProjectsStagePoint(?, ?, ?, ?)");
		$result->bind_param('iiii', $pharmId, $projectId, $stageId, $force);
		$result->execute();

		$result->close();
		
		break;

	case 'GET_FILLING':
		$projectId = $_REQUEST['projectId'];
		$userTypeId = $_SESSION['userTypeId'];
		$employeeId = $_REQUEST['employeeId'];
		$employeeId = $employeeId > 0 ? $employeeId : $p_id;
		$out = '';
		$stages = array();
		$rowNames = array();

		$rows = array();

		$result = $db_connection->prepare("call fillReport_getStageRows(?)");
		$result->bind_param('i', $employeeId);
		$result->execute();
		$result->bind_result($rowName);
		while($result->fetch())
		{
			array_push($rowNames, $rowName);
		}
		$result->close();
		while($db_connection->next_result()) $db_connection->store_result();

		$rows[0] = array();
		array_push($rows[0], count($rowNames));
		array_push($rows[0], 'Сотрудники');
		$rows[1] = array();
		$result = $db_connection->prepare("select a.id, concat(a.name, ' (', DATE_FORMAT(a.startDate, '%d.%m.%Y'), ' - ', DATE_FORMAT(a.endDate, '%d.%m.%Y'), ')') as name from projects_stages a INNER JOIN projects_stageUserTypes tt on tt.projectStagesId = a.id and (tt.userTypesId = (SELECT userTypesId FROM users WHERE id = ?) OR (tt.userTypesId = 1 AND (SELECT userTypesId FROM users WHERE id = ?) IN (3, 4, 6)) OR (SELECT userTypesId FROM users WHERE id = ?) IN (5)) where a.projectsId=? and DATEDIFF(NOW(), a.startDate)>=0");
		$result->bind_param('iiii', $employeeId, $employeeId, $employeeId, $projectId);
		$result->execute();
		$result->bind_result($stageId, $stageName);
		while($result->fetch())
		{
			array_push($stages, array("id" => $stageId, "name" => $stageName));
		}
		$result->close();
		while($db_connection->next_result()) $db_connection->store_result();

		for($idx=0;$idx<count($stages);$idx++)
		{
			$columnNames = array();
			$result = $db_connection->prepare("call fillReport_getStageColumns(?)");
			$result->bind_param('i', $stages[$idx]["id"]);
			$result->execute();
			$result->bind_result($columnName);
			while($result->fetch())
			{
				array_push($rows[1], 1);
				array_push($rows[1], $columnName);
				array_push($columnNames, $columnName);
			}
			$result->close();
			array_push($rows[0], count($columnNames));
			array_push($rows[0], $stages[$idx]["name"]);
			while($db_connection->next_result()) $db_connection->store_result();
		}

		for($idx=0;$idx<count($stages);$idx++)
		{
			$cnt = 2;
			$result = $db_connection->prepare("call fillReport_getStageReport(? ,?)");
			$result->bind_param('ii', $employeeId, $stages[$idx]["id"]);
			$result->execute();
			$meta = $result->result_metadata();
			while ($field = $meta->fetch_field()) 
			{ 
				$params[] = &$row[$field->name];
			}
			call_user_func_array(array($result, 'bind_result'), $params);
			
			while ($result->fetch()) 
			{
				if($idx==0)
				{
					$rows[$cnt] = array();
				}
				$rcnt = 0;
				foreach($row as $key => $val) 
				{
					if($idx==0 || $rcnt>=count($rowNames))
					{
						if($val!='')
							array_push($rows[$cnt],$val);
					}
					$rcnt++;
				}
				$cnt++;
			}
			$result->close();
			
			while($db_connection->next_result()) $db_connection->store_result();
		}

		$out .= '<table class="item">';
		$out .= '<tr>';
		$out .= '<th rowspan=2 colspan='.$rows[0][0].'>'.$rows[0][1].'</th>';
		for($i=2;$i<count($rows[0]);$i+=2)
		{
			$out .= '<th rowspan=1 colspan='.$rows[0][$i].'>'.$rows[0][$i+1].'</th>';
		}
		$out .= '</tr>';
		$out .= '<tr>';
		for($i=0;$i<count($rows[1]);$i+=2)
		{
			$out .= '<th colspan='.$rows[1][$i].'>'.$rows[1][$i+1].'</th>';
		}
		$out .= '</tr>';
		for($i=2;$i<count($rows);$i++)
		{
			$out .= '<tr>';
			for($j=0;$j<count($rows[$i]);$j++)
			{
				$out .= '<td>'.$rows[$i][$j].'</td>';
			}
			$out .= '</tr>';
		}
		$out .= '</table>';
		echo $out;
		
		break;

	case 'GET_FORMHEADER':
		$projectId = $_REQUEST['projectId'];
		$stageId = $_REQUEST['stageId'];
		
		$sql = "call data_getFormHeader (?, ?);";
		$result = $db_connection->prepare($sql);
		$result->bind_param('ii', $projectId, $stageId);
		$result->execute();
		
		$result->bind_result($id, $caption, $field, $editor, $type, $mandatory, $listId);

		$obj = array("columns" => array());
		$obj['fields'] = array();

		array_push($obj['columns'], array(header => 'id'));
		$obj['columns'][count($obj['columns'])-1]['hidden'] = true;
		$obj['columns'][count($obj['columns'])-1]['dataIndex'] = 'id';
		array_push($obj['fields'], array(name => 'id'));
		//$obj['fields'][count($obj['fields'])-1]['type'] = 'integer';
		
		
		array_push($obj['columns'], array(header => 'Бренд'));
		$obj['columns'][count($obj['columns'])-1]['dataIndex'] = 'name';
		array_push($obj['fields'], array(name => 'name'));
		//$obj['fields'][count($obj['fields'])-1]['type'] = 'string';
		
		while ($result->fetch()) {
			array_push($obj['fields'], array(name => $field));
			//$obj['fields'][count($obj['fields'])-1]['type'] = $type;
			
			array_push($obj['columns'], array(header => $caption));
			$obj['columns'][count($obj['columns'])-1]['dataIndex'] = $field;
			$obj['columns'][count($obj['columns'])-1]['width'] = 'auto';
			if ($editor == 1){
				$xtype = ($type == "integer" || $type == "double" || $type == "string" || $type == "numeric" ? "textfield" : $type);
				$obj['columns'][count($obj['columns'])-1]['editor'] = new StdClass();
				$obj['columns'][count($obj['columns'])-1]['editor']->xtype = $xtype;
				$obj['columns'][count($obj['columns'])-1]['editor']->allowBlank = !(bool)$mandatory;
				if ($xtype == "combo"){
					$obj['columns'][count($obj['columns'])-1]['editor']->listId = $listId;
				}
			}
		}
		
		$result->close();
		
		break;

	case 'GET_LIST':
		$listId = $_REQUEST['listId'];
		
		$result = $db_connection->prepare("call getList(?)");
		$result->bind_param('i', $listId);
		$result->execute();
		$result->bind_result($id, $name);
		
		$obj = array("records" => array());	
		
		while($result->fetch())
		{
			array_push($obj['records'], array(rowId => $id, name => $name));
		}
		$result->close();
		break;
}

if (isset($obj)){
	echo json_encode($obj);
}

$db_connection->close();
