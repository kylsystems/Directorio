<?php
require_once("config.php");
require_once("include.php");
require_once("template_index.php");
header("Content-Type: text/html; charset=utf-8"); 

$c *= 1;
if ($c == 0) $c = 1;
$bd = mysql_connect($mysql_hostname, $mysql_user, $mysql_password);
mysql_select_db($mysql_database, $bd);
$sql = mysql_query("SELECT name, title, description, pages, ref FROM {$prefix}categories WHERE id = $c");
$current_category = array_map("parse_output",mysql_fetch_array($sql, MYSQL_ASSOC));
if (($c > 1) & !$current_category["name"]){
	mysql_close();
	header("Location: {$dir}");
	exit();
};
if (!$current_category["name"]) $current_category["name"] = "Free PHP Directory Script";
if (!$current_category["description"]) $current_category["description"] = $current_category["name"]." ".$current_category["title"];
$replace = array("[CATEGORY_NAME]" => $current_category["name"], "[CATEGORY_TITLE]" => $current_category["title"], "[CATEGORY_DESCRIPTION]" => $current_category["description"]);
echo strtr($TEMPLATE["HEADING"],$replace);
flush();
$last_category = false;
$ref = $c;
while(!$last_category){
	$n_parent_categories += 1;
	$sql = mysql_query("SELECT id, name, ref FROM {$prefix}categories WHERE id = $ref");
	$parent_categories[$n_parent_categories-1] = array_map("parse_output",mysql_fetch_array($sql, MYSQL_ASSOC));
	if ($parent_categories[$n_parent_categories-1]["ref"] == 0){
		$last_category = true;
	}else{
		$ref = $parent_categories[$n_parent_categories-1]["ref"];
	};
};
echo $TEMPLATE["PATH"]["HEADING"];
for ($x = $n_parent_categories-1; $x >= 0; $x--){
	if ($x != $n_parent_categories-1) echo $TEMPLATE["PATH"]["SEPARATOR"];
	if ($x == 0){
		$replace = array("[CATEGORY_NAME]" => $parent_categories[$x]["name"]);
		echo strtr($TEMPLATE["PATH"]["CURRENT_CATEGORY"],$replace);
	}else{
		if ($parent_categories[$x]["id"] > 1){
			$category_url = $dir.'index.php?c='.$parent_categories[$x]["id"];
		}else{
			$category_url = $dir;
		};
		$replace = array("[CATEGORY_NAME]" => $parent_categories[$x]["name"], "[CATEGORY_URL]" => $category_url);
		echo strtr($TEMPLATE["PATH"]["CATEGORY"],$replace);
	};
};
echo $TEMPLATE["PATH"]["FOOTER"];
flush();
$sql = mysql_query("SELECT id, name FROM {$prefix}categories WHERE ref = $c ORDER BY name");
$n_subcategories = mysql_num_rows($sql);
for ($x = 0; $x < $n_subcategories; $x++){
	$subcategories[$x] = array_map("parse_output",mysql_fetch_array($sql, MYSQL_ASSOC));
};
if ($n_subcategories > 0){
	$replace = array("[NUMBER_CATEGORIES]" => $n_subcategories, "[CATEGORY_NAME]" => $current_category["name"]);
	echo strtr($TEMPLATE["SUBCATEGORIES"]["HEADING"],$replace);
	echo $TEMPLATE["SUBCATEGORIES"]["BEFORE_COLUMNS"];
	for ($x = 0; $x < ceil($n_subcategories/2); $x++){
		$replace = array("[CATEGORY_NAME]" => $subcategories[$x]["name"], "[CATEGORY_URL]" => $dir.'index.php?c='.$subcategories[$x]["id"]);
		echo strtr($TEMPLATE["SUBCATEGORIES"]["CATEGORY"],$replace);
	};
	echo $TEMPLATE["SUBCATEGORIES"]["BETWEEN_COLUMNS"];
	for ($x = ceil($n_subcategories/2); $x < $n_subcategories; $x++){
		$replace = array("[CATEGORY_NAME]" => $subcategories[$x]["name"], "[CATEGORY_URL]" => $dir.'index.php?c='.$subcategories[$x]["id"]);
		echo strtr($TEMPLATE["SUBCATEGORIES"]["CATEGORY"],$replace);
	};
	echo $TEMPLATE["SUBCATEGORIES"]["AFTER_COLUMNS"];
	echo $TEMPLATE["SUBCATEGORIES"]["FOOTER"];
}else{
	$replace = array("[CATEGORY_NAME]" => $current_category["name"]);
	echo strtr($TEMPLATE["SUBCATEGORIES"]["NO_CATEGORIES"],$replace);
};
if ($current_category["pages"] == "y"){
	$replace = array("[CATEGORY_NAME]" => $current_category["name"], "[SUBMISSION_URL]" => $dir.'add_url.php?c='.$c);
	echo strtr($TEMPLATE["SUBMISSION_LINK"],$replace);
	flush();
	if ($s == 0) $s = 1;
	$n = 10;
	$sql = mysql_query("SELECT COUNT(*) AS total_pages FROM {$prefix}pages WHERE category = $c AND accepted = 'y'");
	$total_pages = mysql_result($sql,0,"total_pages");
	if ($total_pages > 0){
		$sql = mysql_query("SELECT id, url, title, description FROM {$prefix}pages WHERE category = $c AND accepted = 'y' ORDER BY title LIMIT ".($s-1).",$n");
		$n_pages = mysql_num_rows($sql);
		for ($x = 0; $x < $n_pages; $x++){
			$pages[$x] = array_map("parse_output",mysql_fetch_array($sql, MYSQL_ASSOC));
		};
	};
	$e = min($s + $n - 1, $s + $n_pages - 1);
	if ($n_pages > 0){
		$replace = array("[STARTING_PAGE_NUMBER]" => $s, "[ENDING_PAGE_NUMBER]" => $e, "[TOTAL_PAGES]" => $total_pages, "[CATEGORY_NAME]" => $current_category["name"]);
		echo strtr($TEMPLATE["PAGES"]["HEADING"],$replace);
		for ($x = 0; $x < $n_pages; $x++){
			if ($pages[$x]["domain"]){
				$replace = array("[PAGE_NUMBER]" => $x, "[PAGE_TITLE]" => $pages[$x]["title"], "[PAGE_DESCRIPTION]" => $pages[$x]["description"], "[PAGE_URL]" => $pages[$x]["url"], "[PAGE_DOMAIN]" => $pages[$x]["domain"]);
				echo strtr($TEMPLATE["PAGES"]["FEED_PAGE"],$replace);
			}else{
				$replace = array("[PAGE_NUMBER]" => $x, "[PAGE_TITLE]" => $pages[$x]["title"], "[PAGE_DESCRIPTION]" => $pages[$x]["description"], "[PAGE_URL]" => $pages[$x]["url"], "[PAGE_DOMAIN]" => $pages[$x]["domain"]);
				echo strtr($TEMPLATE["PAGES"]["PAGE"],$replace);
			};
		};
		if ($s != 1 || $e != $total_pages){
			function pagination($s){
				global $c, $dir;
				if ($c != 1) { $query = "index.php?c=$c"; };
				if (($c != 1) & $s != 1){
					$query .= "&s=$s";
				}elseif ($s != 1){
					$query = "index.php?s=$s";
				};
				return $dir.$query;
			};
			echo $TEMPLATE["PAGES"]["PAGINATION"]["HEADING"];
			if ($s != 1){
				$previous = $s - $n;
				$replace = array("[PAGINATION_URL]" => pagination($previous));
				echo strtr($TEMPLATE["PAGES"]["PAGINATION"]["PREVIOUS"],$replace);
			};
			for ($x = 1; $x <= ceil($total_pages/$n); $x++){
				$current = ($x-1) * $n + 1;
				if ($current == $s){
					$replace = array("[PAGINATION_NUMBER]" => $x);
					echo strtr($TEMPLATE["PAGES"]["PAGINATION"]["CURRENT_NUMBER"],$replace);
				}else{
					$replace = array("[PAGINATION_NUMBER]" => $x, "[PAGINATION_URL]" => pagination($current));
					echo strtr($TEMPLATE["PAGES"]["PAGINATION"]["NUMBER"],$replace);
				};
			};
			if ($e < $total_pages){
				$next = $s + $n;
				$replace = array("[PAGINATION_URL]" => pagination($next));
				echo strtr($TEMPLATE["PAGES"]["PAGINATION"]["NEXT"],$replace);
			};
			echo $TEMPLATE["PAGES"]["PAGINATION"]["FOOTER"];
		};
		echo $TEMPLATE["PAGES"]["FOOTER"];
	}else{
		$replace = array("[CATEGORY_NAME]" => $current_category["name"]);
		echo strtr($TEMPLATE["PAGES"]["NO_PAGES"],$replace);
	};
};

$sponsor = get_sponsor();

$replace = array("[SPONSOR_URL]" => $sponsor["link"], "[SPONSOR_TEXT]" => $sponsor["text"]);
echo strtr($TEMPLATE["FOOTER"],$replace);
?>
