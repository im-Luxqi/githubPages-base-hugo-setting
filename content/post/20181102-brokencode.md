---
title: Excel导入导出（POI）
date: 2018-11-02
tags: ["碎片代码"]
---

记录一些工作中零碎的代码

<!--more-->


#### 一、Excel导入List,完成验证
<a href="/img/broken_code/file1.xls" download="360度考评问卷模板.xls">导入的excel格式</a>
{{< highlight javascript "linenos=inline">}}
/**
 * 解析excel 并验证
 * @param inputStream 文件流
 * @param fileName 文件名
 * @param surveyId  具体参数
 * @param retData   json 返回集合
 * @return
 */
private List<Map<String, Object>> findServeyExcelData(
        InputStream inputStream,String fileName,String surveyId, Data retData){
    List<Map<String, Object>> excelList = new ArrayList<>();
    //拿到excle数据
    Workbook workbook = null;
    try {
        workbook = CigExcel.getWorkbook(inputStream, fileName);
    } catch (Exception e) {
        e.printStackTrace();
    }

    /**excel，以及数据库表的一些基本情况
     *       sheet1：SURVEY_TYPE = 0301（处级评价科级）
     *       sheet2：SURVEY_TYPE = 0202（科级互评（含自评））
     *       sheet3：SURVEY_TYPE = 0201（科级评价科员）
     *       sheet4：SURVEY_TYPE = 0102（科员互评（含自评））
     *       sheet5：SURVEY_TYPE = 0103（科员评价科级）
     */
    final String[] sheetTypeCode = new String[] { "0301","0202","0201","0102","0103"};
    final String[] sheetTypetName = new String[] { "处级评价科级","科级互评（含自评）"
                                     ,"科级评价科员","科员互评（含自评）","科员评价科级"};
    final String[] titleArr = new String[] { "序号", "名称", "含义", "评分标准"};
    final String[] titleCode = new String[] { "NO", "NAME", "DETAIL", "FEN_TYPE_CODE"};
    final int validSheetMaxNum = 5;


    //循环每一个sheet,每一行，每一列
    sheetFor:
    for (int i = 0; i < validSheetMaxNum; i++) {
        Sheet sheet = workbook.getSheetAt(i);
        int rowNum = sheet.getPhysicalNumberOfRows();
        int cellNum = sheet.getRow(1).getPhysicalNumberOfCells();
        for (int rowIndex = 0; rowIndex < rowNum; rowIndex++) {
            if (rowIndex > 0) {
                Row row = sheet.getRow(rowIndex);
                if (row == null || (CigExcel.initCellVal(row.getCell(1)) == null &&
                        CigExcel.initCellVal(row.getCell(2)) == null)) {
                    continue;
                }
                String[] strArr = new String[titleArr.length];// 得到列对象
                // 将每一行的每一列对象取出来放入map
                Map<String, Object> map = new HashMap<>();
                for (int cellIndex = 0; cellIndex < cellNum; cellIndex++) {
                    Cell cell = row.getCell(cellIndex);
                    if (cell == null) {
                        continue;
                    }
                    try {
                        strArr[cellIndex] = CigExcel.initCellVal(cell);// 检查数据类型
                        map.put(titleCode[cellIndex], strArr[cellIndex]);
                        //检查字段

                    } catch (Exception e) {
                        logger.debug("录入数据异常：" + e.getMessage());
                        continue;
                    }
                }
                map.put("SURVEY_TYPE",sheetTypeCode[i]);//考评名称
                map.put("SURVEY_ID",surveyId);//问卷Id

                //导入失败，系统提示“sheet页第X行X字段不符合要求，请重新导入！”
                /**验证各数据
                 *      序号：必填；数字，整数
                 *      名称：必填；最多80个汉字
                 *      含义：必填；最多340个汉字
                 *      评分标准：必填；数字，4位整数；有效性检测SQL：
                 */
                boolean checkMark = true;
                Object no = map.get("NO");
                Object name = map.get("NAME");
                Object detail = map.get("DETAIL");
                Object fen_type_code = map.get("FEN_TYPE_CODE");
                if(checkMark && ( no == null ||
                        no !=null && !((String) no).matches("[0-9]*"))){// 序号必须为整数
                    retData.add("isOk", false);
                    retData.add("msg", sheetTypetName[i] +
                            "第"+ (rowIndex+1) +"行; " +
                            " \"序号字段\" " +
                            "不符合要求，请重新导入！");
                    checkMark = false;
                }
                if(checkMark && (name == null ||
                        name !=null && name.toString().length()>80)) {// 名称最多80个汉字
                    retData.add("isOk", false);
                    retData.add("msg", sheetTypetName[i] +
                            "第"+ (rowIndex+1) +"行; " +
                            " \"名称字段\" " +
                            "不符合要求，请重新导入！");
                    checkMark = false;
                }
                if(checkMark && (detail == null ||
                        detail !=null && detail.toString().length()>340)) {// 含义最多340个汉字
                    retData.add("isOk", false);
                    retData.add("msg", sheetTypetName[i] +
                            "第"+ (rowIndex+1) +"行; " +
                            " \"含义字段\" " +
                            "不符合要求，请重新导入！");
                    checkMark = false;
                }

                if(checkMark && ( fen_type_code==null ||
                        fen_type_code !=null && !((String) fen_type_code).matches("\\d{4}") ||
                        fen_type_code !=null &&
                                !staffsWorkNotesDao.validSurveyType((String) fen_type_code))
                    ) {//评分标准为四位整数，并有意义
                    retData.add("isOk", false);
                    retData.add("msg", sheetTypetName[i] +
                            "第"+ (rowIndex+1) +"行；" +
                            " \"评分标准字段\" " +
                            "不符合要求，请重新导入！");
                    checkMark = false;
                }
                if (!checkMark) {
                    excelList.clear();
                    break sheetFor;
                }
                excelList.add(map);
            }
        }
    }
    return excelList;
}

{{</ highlight >}}
#### 二、Excel导出（速度不敢恭维）
<a href="/img/broken_code/file2.xlsx" download="商业销售导出.xlsx">导入的excel格式</a>
{{< highlight javascript "linenos=inline">}}
/**
 * 通用的导出Excel类，（Excel 2007 OOXML (.xlsx)格式 ）
 * dataList里的每一个Object数组一个元素
 */
public class ExportExcelUtil {
	static final Logger logger = LoggerFactory.getLogger(ExportExcelUtil.class);
	
	private List<Object[]>  dataList = new ArrayList<Object[]>(); // 对象数组的List集合

	public ExportExcelUtil(List<Object[]> dataList) {
		this.dataList = dataList;
	}

	// 导出数据
	public String exportData(){
		ZipSecureFile.setMinInflateRatio(0);
		SimpleDateFormat yyyyMM = new SimpleDateFormat("yyyyMM");
		SimpleDateFormat yyyyMMChina = new SimpleDateFormat("yyyy年MM月");
		SimpleDateFormat MMdd = new SimpleDateFormat("MMdd");

		//本周周末日期，下周周末日期
		Calendar calendarTime = Calendar.getInstance();
        int day_of_week = calendarTime.get(Calendar.DAY_OF_WEEK) - 1;
        if (day_of_week == 0) day_of_week = 7;
        calendarTime.add(Calendar.DATE, -day_of_week + 6);
		Date thisTime = calendarTime.getTime();
		calendarTime.add(Calendar.DAY_OF_YEAR,7);
		Date nextTime = calendarTime.getTime();

		//声明一个工作薄 Excel 2007 OOXML (.xlsx)格式，创建一个sheet
		SXSSFWorkbook workbook = new SXSSFWorkbook();
		String thisWeekyyMM = MMdd.format(thisTime);
		String nextWeekyyMM = MMdd.format(nextTime);
		String sheetName = yyyyMMChina.format(new Date());
		SXSSFSheet sheet = workbook.createSheet(sheetName);

		// sheet样式定义(columnTopStyle：头样式，columnStyle：标题样式，style：单元格样式)
		CellStyle columnTopStyle = this.getColumnTopStyle(workbook,14);
		CellStyle columnStyle = this.getColumnStyle(workbook,12);
		CellStyle style = this.getStyle(workbook,11);

		/**创建复杂表头
		 *    excelHeader0: 第一行表头字段（跨行字段重复）
		 *    headnum0：  “0,1,0,0”  ===>  “起始行，截止行，起始列，截止列”
		 */
		String[] excelHeader0 = { "省份", "地市", "品牌","规格",
				"最新库存", "最新库存","最新库存",
				"本月实际","本月实际","本月实际","本月实际",
				"本年累计","本年累计","本年累计","本年累计",
				"本月预测","本月预测","本月预测","本月预测","本月预测","本月预测",
				"本周"+thisWeekyyMM,"本周"+thisWeekyyMM,"本周"+thisWeekyyMM, 
				"下周"+nextWeekyyMM,"下周"+nextWeekyyMM,"下周"+nextWeekyyMM,
		};
		String[] headnum0 = { "0,1,0,0", "0,1,1,1", "0,1,2,2","0,1,3,3", "0,0,4,6",
				"0,0,7,10", "0,0,11,14", "0,0,15,20", "0,0,21,23", "0,0,24,26"};
		String[] excelHeader1 = {"","","","", "数据日期", "商业库存", "商业存销比",
				"销量", "同期", "同比量","同比幅",
				"销量", "同期", "同比量","同比幅",
				"销量", "同期", "同比量","同比幅","月底库存","月底存销比",
				"销量", "库存", "存销比",
				"销量", "库存", "存销比",
		};
		// 第一行表头
		SXSSFRow rowHeader = sheet.createRow(0);
		for (int i = 0; i < excelHeader0.length; i++) {
			SXSSFCell cell = rowHeader.createCell(i);
			cell.setCellValue(excelHeader0[i]);
			cell.setCellStyle(columnStyle);

				for (int j = 0; j < excelHeader0.length; j++) {
					cell = rowHeader.createCell(j);
					cell.setCellValue(excelHeader0[j]);
					cell.setCellStyle(columnStyle);
				}
		}

		// 动态合并单元格
		for (int i = 0; i < headnum0.length; i++) {
			String[] temp = headnum0[i].split(",");
			Integer startrow = Integer.parseInt(temp[0]);
			Integer overrow = Integer.parseInt(temp[1]);
			Integer startcol = Integer.parseInt(temp[2]);
			Integer overcol = Integer.parseInt(temp[3]);
			sheet.addMergedRegion(
					new CellRangeAddress(startrow, overrow, startcol, overcol));
		}
		// 第二行表头
		rowHeader = sheet.createRow(1);
		for (int i = 0; i < excelHeader1.length; i++) {
			SXSSFCell cell = rowHeader.createCell(i);
			cell.setCellValue(excelHeader1[i]);
			cell.setCellStyle(columnStyle);
		}

		//根据列名设置每一列的宽度
		for(int i = 1;i<excelHeader0.length;i++){
			int length = excelHeader0[i].toString().length();
			int width = 2*(length+3)*256;
			if(i==3){//第三列数据要长点
				width = 2*(length+7)*256;
			}
			sheet.setColumnWidth(i,width );
		}

		// 产生其它行（将数据列表设置到对应的单元格中）
		for (int i = 0; i < dataList.size(); i++) {
			Object[] obj = dataList.get(i);
			SXSSFRow row = sheet.createRow(i+2);
			row.setHeightInPoints(17.25f);
			for (int j = 0; j < obj.length; j++) {
				SXSSFCell  cell = row.createCell(j,SXSSFCell.CELL_TYPE_STRING);
					 if(!"".equals(obj[j]) && obj[j] != null){//设置单元格的值
						 cell.setCellValue(obj[j].toString());
					 }else{
					 	cell.setCellValue("  "); }
					 if(j==2 || j==3) {//第四列（规格）左对齐，其他列居中
						 CellStyle leftStyle = this.getStyle(workbook,11);
						 leftStyle.setAlignment(CellStyle.ALIGN_LEFT);
						 cell.setCellStyle(leftStyle);
					 }else{
						 cell.setCellStyle(style);}
			}
		}
		//文件路径
		String fileName = SysHelper.get32UUIDStr() + ".xlsx";
		String rootpath = ConstantDefine.uploadFilePath;
		String relativePath =  "/mobile/marketing/sale/";
		String fullPath = rootpath + relativePath;
		IOKit.creDirIfNotExsit(fullPath);
		fullPath += fileName ;

		if(workbook !=null){// 输出到服务器上
			FileOutputStream fileOutputStream = null;
			try {
				fileOutputStream = new FileOutputStream(fullPath);
				workbook.write(fileOutputStream);
				fileOutputStream.close();
			} catch (Exception e) {
				logger.error("商业销售导出excel生成失败");
				e.printStackTrace();
			}

		}

		return relativePath + fileName;
    }

	public CellStyle getColumnTopStyle(SXSSFWorkbook workbook,int fontSize) {  
        // 设置字体
        Font font = workbook.createFont();
        //设置字体大小
        font.setFontHeightInPoints((short)fontSize);
        //字体加粗
        font.setBoldweight(Font.BOLDWEIGHT_BOLD);
        //设置字体名字 
        font.setFontName("宋体");
        //设置样式;
        CellStyle style = workbook.createCellStyle();
        //在样式用应用设置的字体;    
        style.setFont(font);
        //设置自动换行;
        style.setWrapText(false);
        //设置水平对齐的样式为居中对齐;
        style.setAlignment(CellStyle.ALIGN_CENTER);
        //设置垂直对齐的样式为居中对齐;
        style.setVerticalAlignment(CellStyle.VERTICAL_CENTER);
        return style;
	}  
	
	public CellStyle getColumnStyle(SXSSFWorkbook workbook,int fontSize) {  
        // 设置字体
		Font font = workbook.createFont();
        //设置字体大小
        font.setFontHeightInPoints((short)fontSize);
        //字体加粗
        font.setBoldweight(Font.BOLDWEIGHT_BOLD);
        //设置字体名字 
        font.setFontName("宋体");
        //设置样式;
        CellStyle style = workbook.createCellStyle();
        //设置底边框;
        style.setBorderBottom(CellStyle.BORDER_THIN);
        //设置底边框颜色;
        style.setBottomBorderColor(HSSFColor.BLACK.index);
        //设置左边框;
        style.setBorderLeft(CellStyle.BORDER_THIN);
        //设置左边框颜色;
        style.setLeftBorderColor(HSSFColor.BLACK.index);
        //设置右边框;
        style.setBorderRight(CellStyle.BORDER_THIN);
        //设置右边框颜色;
        style.setRightBorderColor(HSSFColor.BLACK.index);
        //设置顶边框;
        style.setBorderTop(CellStyle.BORDER_THIN);
        //设置顶边框颜色;
        style.setTopBorderColor(HSSFColor.BLACK.index);
        //在样式用应用设置的字体;    
        style.setFont(font);
        //设置自动换行;
        style.setWrapText(false);
        //设置水平对齐的样式为居中对齐;
        style.setAlignment(CellStyle.ALIGN_CENTER);
        //设置垂直对齐的样式为居中对齐;
        style.setVerticalAlignment(CellStyle.VERTICAL_CENTER);
        
        //设置背景填充色（前景色）
        style.setFillForegroundColor(HSSFColor.LIGHT_CORNFLOWER_BLUE.index);
        style.setFillPattern(CellStyle.SOLID_FOREGROUND);
        return style;
	}
	
	public CellStyle getStyle(SXSSFWorkbook workbook,int fontSize) {
        //设置字体
        Font font = workbook.createFont();
        //设置字体大小
        font.setFontHeightInPoints((short)fontSize);
        //字体加粗  
        //font.setBoldweight(Font.BOLDWEIGHT_BOLD);  
        //设置字体名字   
        font.setFontName("宋体");
        //设置样式;
        CellStyle style = workbook.createCellStyle();
        //设置底边框;
        style.setBorderBottom(CellStyle.BORDER_THIN);
        //设置底边框颜色;
        style.setBottomBorderColor(HSSFColor.BLACK.index);
        //设置左边框;     
        style.setBorderLeft(CellStyle.BORDER_THIN);
        //设置左边框颜色;   
        style.setLeftBorderColor(HSSFColor.BLACK.index);
        //设置右边框;   
        style.setBorderRight(CellStyle.BORDER_THIN);
        //设置右边框颜色;   
        style.setRightBorderColor(HSSFColor.BLACK.index);
        //设置顶边框;   
        style.setBorderTop(CellStyle.BORDER_THIN);
        //设置顶边框颜色;    
        style.setTopBorderColor(HSSFColor.BLACK.index);
        //在样式用应用设置的字体;
        style.setFont(font);
        //设置自动换行;   
        style.setWrapText(false);
        //设置水平对齐的样式为居中对齐;
        style.setAlignment(CellStyle.ALIGN_CENTER);
        //设置垂直对齐的样式为居中对齐;
        style.setVerticalAlignment(CellStyle.VERTICAL_CENTER);
         
        return style;
	}

}

{{</ highlight >}}