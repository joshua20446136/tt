import com.ctop.base.repository.BaseCompanyRepository;
import com.ctop.degson.dataInterface.dto.SapMrDto;
import com.ctop.degson.dataInterface.dto.SapPrDto;
import com.ctop.degson.dataInterface.dto.SapPrdDto;
import com.ctop.degson.dataInterface.service.SapSendDataService;
import com.ctop.degson.dataInterface.service.WmBizOutFeedbackService;
import com.ctop.fw.common.utils.ListUtil;
import com.ctop.fw.common.utils.StringUtil;
import com.ctop.fw.hr.entity.HrDepartment;
import com.ctop.fw.hr.repository.HrDepartmentRepository;
import com.ctop.interfaces.dto.OutSendDataDto;
import com.ctop.interfaces.dto.SendDataResultDto;
import com.ctop.interfaces.service.ISendDataService;
import com.ctop.les.biz.dto.BizPurchaseOrderDetailDto;
import com.ctop.les.biz.dto.BizPurchaseOrderDto;
import com.ctop.les.biz.service.BizPurchaseOrderDetailService;
import com.ctop.les.biz.service.BizPurchaseOrderService;
import com.ctop.wms.base.entity.WmWarehouse;
import com.ctop.wms.base.repository.WmWarehouseRepository;
import com.ctop.wms.biz.entity.WmBizMrOrderDetail;
import com.ctop.wms.biz.entity.WmBizOut;
import com.ctop.wms.biz.entity.WmBizOutDetail;
import com.ctop.wms.biz.repository.WmBizMrOrderDetailRepository;
import com.ctop.wms.biz.repository.WmBizOutDetailRepository;
import com.ctop.wms.biz.repository.WmBizOutRepository;
import com.ctop.wms.core.event.LabelOutEvent;
import com.ctop.wms.gs.dto.GsScrapOrderDetailDto;
import com.ctop.wms.gs.entity.GsScrapOrder;
import com.ctop.wms.gs.repository.GsScrapOrderRepository;
import com.ctop.wms.gs.service.GsScrapOrderDetailService;
import com.ctop.wms.wm.dto.WmSkuLabelDto;
import com.ctop.wms.wm.entity.WmSkuLabel;
import com.ctop.wms.wm.repository.WmSkuLabelRepository;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import javax.persistence.EntityManager;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

@Service
@Transactional
public class WmBizOutFeedbackService implements ISendDataService {
  @Value("${erpUrl}")
  private String erpUrl;
  
  @Autowired
  private WmSkuLabelRepository wmSkuLabelRepository;
  
  @Autowired
  private EntityManager entityManager;
  
  @Autowired
  private WmBizOutRepository wmBizOutRepository;
  
  @Autowired
  private SapSendDataService sapSendDataService;
  
  @Autowired
  private WmWarehouseRepository wmWarehouseRepository;
  
  @Autowired
  private BaseCompanyRepository baseCompanyRepository;
  
  @Autowired
  private BizPurchaseOrderService bizPurchaseOrderService;
  
  @Autowired
  private BizPurchaseOrderDetailService bizPurchaseOrderDetailService;
  
  @Autowired
  private GsScrapOrderRepository gsScrapOrderRepository;
  
  @Autowired
  private GsScrapOrderDetailService gsScrapOrderDetailService;
  
  @Autowired
  private WmBizOutDetailRepository wmBizOutDetailRepository;
  
  @Autowired
  private WmBizMrOrderDetailRepository wmBizMrOrderDetailRepository;
  
  @Autowired
  private HrDepartmentRepository hrDepartmentRepository;
  
  @Async
  @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
  public void sendWmBizOutInfo(LabelOutEvent event) {
    WmSkuLabelDto wmSkuLabeldto = event.getWmSkuLabel();
    WmSkuLabel wmSkuLabel = (WmSkuLabel)this.wmSkuLabelRepository.findOne(wmSkuLabeldto.getWslUuid());
    WmBizOut wmBizOut = (WmBizOut)this.wmBizOutRepository.findOne(wmSkuLabel.getWboUuid());
    if (StringUtil.equals("return", wmBizOut.getBizType())) {
      poReturn(wmSkuLabel, wmBizOut);
    } else if (StringUtil.equals("scrap", wmBizOut.getBizType())) {
      scrap(wmSkuLabel, wmBizOut);
    } else if (StringUtil.equals("mr", wmBizOut.getFromType())) {
      mrOff(wmSkuLabel, wmBizOut);
    } 
  }
  
  public void mrOff(WmSkuLabel wmSkuLabel, WmBizOut wmBizOut) {
    WmBizOutDetail wmBizOutDetail = (WmBizOutDetail)this.wmBizOutDetailRepository.findOne(wmSkuLabel.getWbodUuid());
    if (!StringUtil.equals("Y", wmBizOutDetail.getPrintExt10())) {
      WmBizMrOrderDetail wbmod = (WmBizMrOrderDetail)this.wmBizMrOrderDetailRepository.findOne(wmBizOutDetail.getFromDetailUuid());
      SapMrDto sapDto = new SapMrDto();
      String WERKS = "";
      WmWarehouse warehouse = (WmWarehouse)this.wmWarehouseRepository.findOne(wmSkuLabel.getWarehouseUuid());
      if (StringUtil.isNotEmpty(warehouse.getDepartmentUuid())) {
        HrDepartment hrDepartment = (HrDepartment)this.hrDepartmentRepository.findOne(warehouse.getDepartmentUuid());
        if (hrDepartment != null)
          WERKS = hrDepartment.getDepartmentNo(); 
      } 
      String ZFLDH = wmBizOut.getFromNo();
      String ZFLHH = "";
      String MATNR = wmSkuLabel.getSkuCode();
      String ERFMG = wmBizOutDetail.getTtlNum().toString();
      String MEINS = "";
      String AUFNR = "";
      String CHARG = wmSkuLabel.getVndrBatchCode();
      if (StringUtil.isNotEmpty(wbmod)) {
        ZFLHH = wbmod.getExt1();
        MEINS = wbmod.getExt2();
        AUFNR = wbmod.getExt6();
      } 
      sapDto.setWERKS(WERKS);
      sapDto.setZFLDH(ZFLDH);
      sapDto.setZFLHH(ZFLHH);
      sapDto.setMATNR(MATNR);
      sapDto.setERFMG(ERFMG);
      sapDto.setMEINS(MEINS);
      sapDto.setAUFNR(AUFNR);
      sapDto.setCHARG(CHARG);
      wmBizOutDetail.setPrintExt10("Y");
      this.wmBizOutDetailRepository.saveAndFlush(wmBizOutDetail);
      this.sapSendDataService.mrSend(sapDto, wmSkuLabel);
    } 
  }
  
  public void poReturn(WmSkuLabel wmSkuLabel, WmBizOut wmBizOut) {
    Date currentTime = new Date();
    SimpleDateFormat formatter = new SimpleDateFormat("yyyyMMdd HH:mm:ss");
    String BUDAT_MKPF = formatter.format(currentTime);
    SapPrDto sapDto = new SapPrDto();
    List<SapPrdDto> details = new ArrayList<>();
    String REFDOC = wmBizOut.getBizRefCode();
    sapDto.setUuid(wmSkuLabel.getWslUuid());
    sapDto.setEBELN(REFDOC);
    String BLDAT = "";
    String WERKS = "";
    String LGORT = "";
    WmWarehouse wmWarehouse = (WmWarehouse)this.wmWarehouseRepository.findOne(wmSkuLabel.getWarehouseUuid());
    if (wmWarehouse != null)
      LGORT = wmWarehouse.getWarehouseCode(); 
    String EBELP = "";
    String ERFME = "";
    WmWarehouse warehouse = (WmWarehouse)this.wmWarehouseRepository.findOne(wmSkuLabel.getWarehouseUuid());
    if (StringUtil.isNotEmpty(warehouse.getDepartmentUuid())) {
      HrDepartment hrDepartment = (HrDepartment)this.hrDepartmentRepository.findOne(warehouse.getDepartmentUuid());
      if (hrDepartment != null)
        WERKS = hrDepartment.getDepartmentNo(); 
    } 
    BizPurchaseOrderDto po = this.bizPurchaseOrderService.getOrderByPoCode(REFDOC, wmSkuLabel.getCompanyUuid());
    if (po != null) {
      List<BizPurchaseOrderDetailDto> pods = this.bizPurchaseOrderDetailService.queryByPoUuid(po.getPoUuid());
      for (BizPurchaseOrderDetailDto pod : pods) {
        if (StringUtil.equals(wmSkuLabel.getSkuUuid(), pod.getSkuUuid())) {
          EBELP = pod.getSeqNo().toString();
          ERFME = pod.getUnit();
        } 
      } 
    } 
    sapDto.setBLDAT(BUDAT_MKPF);
    sapDto.setBUDAT_MKPF(BUDAT_MKPF);
    SapPrdDto detail = new SapPrdDto();
    String MATNR = wmSkuLabel.getSkuCode();
    detail.setEBELP(EBELP);
    detail.setERFME(ERFME);
    detail.setMATNR(MATNR);
    detail.setWERKS(WERKS);
    detail.setLGORT(LGORT);
    String ERFMG = wmSkuLabel.getBoxQty().toString();
    detail.setERFMG(ERFMG);
    String BWART = "101";
    detail.setBWART(BWART);
    String CHARG = "";
    CHARG = wmSkuLabel.getVndrBatchCode();
    detail.setCHARG(CHARG);
    String INSMK = "";
    if (StringUtil.equals("Y", wmSkuLabel.getNeedQc()))
      INSMK = "2"; 
    detail.setINSMK(INSMK);
    details.add(detail);
    sapDto.setDetail(details);
    this.sapSendDataService.poSend(sapDto, wmSkuLabel);
  }
  
  public void scrap(WmSkuLabel wmSkuLabel, WmBizOut wmBizOut) {
    GsScrapOrder gsScrapOrder = (GsScrapOrder)this.gsScrapOrderRepository.findOne(wmBizOut.getBizRefUuid());
    List<GsScrapOrderDetailDto> gsods = this.gsScrapOrderDetailService.findByGsoUuid(wmBizOut.getBizRefUuid());
    Date currentTime = new Date();
    SimpleDateFormat formatter = new SimpleDateFormat("yyyyMMdd HH:mm:ss");
    SapPrDto sapDto = new SapPrDto();
    List<SapPrdDto> details = new ArrayList<>();
    String REFDOC = gsScrapOrder.getGsoCode();
    String BLDAT = formatter.format(currentTime);
    String BUDAT_MKPF = formatter.format(currentTime);
    sapDto.setREFDOC(REFDOC);
    sapDto.setBLDAT(BLDAT);
    sapDto.setBUDAT_MKPF(BUDAT_MKPF);
    SapPrdDto detail = new SapPrdDto();
    String ZEILE = "";
    String MATNR = wmSkuLabel.getSkuCode();
    String WERKS = "";
    WmWarehouse warehouse = (WmWarehouse)this.wmWarehouseRepository.findOne(wmSkuLabel.getWarehouseUuid());
    if (StringUtil.isNotEmpty(warehouse.getDepartmentUuid())) {
      HrDepartment hrDepartment = (HrDepartment)this.hrDepartmentRepository.findOne(warehouse.getDepartmentUuid());
      if (hrDepartment != null)
        WERKS = hrDepartment.getDepartmentNo(); 
    } 
    String LGORT = "";
    WmWarehouse wmWarehouse = (WmWarehouse)this.wmWarehouseRepository.findOne(wmSkuLabel.getWarehouseUuid());
    if (wmWarehouse != null)
      LGORT = wmWarehouse.getWarehouseCode(); 
    String ERFMG = wmSkuLabel.getInventoryQty().toString();
    String VRKME = "";
    String SPEC_STOCK = "";
    String VAL_SALES_ORD = "";
    String VAL_S_ORD_ITEM = "";
    String SALES_ORD = "";
    String S_ORD_ITEM = "";
    if (StringUtil.isNotEmpty(wmSkuLabel.getMaterialPrjCode())) {
      String[] array = wmSkuLabel.getMaterialPrjCode().split("-");
      SALES_ORD = array[0];
      S_ORD_ITEM = array[array.length - 1];
    } 
    String BWART = "Y11";
    String CHARG = wmSkuLabel.getVndrBatchCode();
    String INSMK = "";
    if (StringUtil.equals("Y", wmSkuLabel.getNeedQc()))
      INSMK = "2"; 
    String KOSTL = "";
    List<GsScrapOrderDetailDto> checkedList = ListUtil.findList(gsods, e -> StringUtil.equals(e.getWslUuid(), wmSkuLabel.getWslUuid()));
    if (ListUtil.isNotEmpty(checkedList))
      KOSTL = ((GsScrapOrderDetailDto)checkedList.get(0)).getBccCode(); 
    String GRUND = gsScrapOrder.getRemark();
    detail.setZEILE(ZEILE);
    detail.setMATNR(MATNR);
    detail.setWERKS(WERKS);
    detail.setLGORT(LGORT);
    detail.setERFMG(ERFMG);
    detail.setVRKME(VRKME);
    detail.setSPEC_STOCK(SPEC_STOCK);
    detail.setVAL_SALES_ORD(VAL_SALES_ORD);
    detail.setVAL_S_ORD_ITEM(VAL_S_ORD_ITEM);
    detail.setSALES_ORD(SALES_ORD);
    detail.setS_ORD_ITEM(S_ORD_ITEM);
    detail.setBWART(BWART);
    detail.setCHARG(CHARG);
    detail.setINSMK(INSMK);
    detail.setKOSTL(KOSTL);
    detail.setGRUND(GRUND);
    details.add(detail);
    sapDto.setDetail(details);
    this.sapSendDataService.scarpSend(sapDto, wmSkuLabel);
  }
  
  public SendDataResultDto sendToData(OutSendDataDto outSendDataDto) {
    return null;
  }
}
