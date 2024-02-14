next js blog 샘플 언젠가는 ~~~ 

https://github.com/safak/next-blog/tree/completed
'use client';

import React, { useEffect, useState, useRef, useCallback } from 'react';
import PropTypes from 'prop-types';

import { fetchDelete, fetchGet } from '@/core/api/network';
import { getTotalPageString, hasValidValue } from '@/core/utils/common';

import { Button, Form, Input, Pagination, Popconfirm, Select, notification } from 'antd';
import styles from '/styles/directory/accession/gasAccession/gasAccession.module.css';

import GridWrapper from '@/components/common/grid/GridWrapper';
import ContentsWrapper from '@/components/common/basic/ContentsWrapper';
import FormBody from '@/components/common/forms/FormBody';
import FormItem from '@/components/common/forms/FormItem';
import DataGrid from '@/components/common/grid/DataGrid';
import GridButton from '@/components/common/grid/GridButton';
import { minusIcon, plusIcon, refreshIcon, searchIcon,filePenIcon, fileTextIcon } from '@/core/utils/icons';
import { getSelectOptions } from '@/core/utils/code';
import NotifyOrganizationSearchModal from '../transferList/NotifyOrganizationSearhModal';

GasAccessionLedger.propTypes = {
  handleMoveToGasAccessionArchive: PropTypes.func,
  activeTabKey: PropTypes.string
};

const gridOptions = {
  pagination: false,
  singleClickEdit: false,
  suppressMovableColumns: true,
  suppressClickEdit: true,
  suppressRowDrag: true,
  suppressRowClickSelection: true,
};

export default function GasAccessionLedger({ handleMoveToGasAccessionArchive, activeTabKey } ) {

  const [ drawer, setDrawer ] = useState(false);
  const handleClickOpenDrawer = () => {
    setDrawer(true);
  };
  const handleClickCloseDrawer = () => {
    setDrawer(false);
  };

  const [ messageApi, contextHolder ] = notification.useNotification();
  const gasAccessionLedgerGridRef = useRef();
  const [ gasAccessionLedgerList, setGasAccessionLedgerList ] = useState([]);
  const [ selectedRowKey, setSelectedRowKey ] = useState([]);
  const [ isModify, setIsModify ] = useState(false);
  const [ inputDisabled, setInputDisabled ] = useState(true);
  const [ accessionState, setAccessionState ] = useState([]);

  const [ gasAccessionLedgerColumnDefs ] = useState([
    {
      headerName: 'No',
      field: 'rowNum',
      //checkboxSelection: true,
      //headerCheckboxSelection: true,
      width: 100,
      cellClass: styles['centered-cell'],
    },
    {
      headerName: '인수기관',
      field: 'tfrOrgNm',
      flex: 1,
      cellClass: styles['centered-cell'],
    },
    {
      headerName: '이관기관',
      field: 'accOrgNm',
      flex: 1,
      cellClass: styles['centered-cell'],
    },
    {
      headerName: '이관년도',
      field: 'tfrYear',
      flex: 1,
      cellClass: styles['centered-cell'],
    },
    {
      headerName: '흐므륵',
      field: 'fundNm',
      flex: 1,
      cellClass: 'ag-clickable-cell',
      onCellClicked: cellParam => handleClickColumn(cellParam.data,'fundNm')
    },
    {
      headerName: '당스',
      field: 'ldgrNm',
      flex: 1,
      cellClass: 'ag-clickable-cell',
      onCellClicked: cellParam => handleClickColumn(cellParam.data,'ldgrNm')
    },
    {
      headerName: '닉츠수',
      field: 'archQuantity',
      flex: 1,
      cellClass: styles['centered-cell'],
    },
    {
      headerName: '베름트수',
      field: 'docQuantity',
      flex: 1,
      cellClass: styles['centered-cell'],
    },
    {
      headerName: '이관상태',
      field: 'tfrStsNm',
      flex: 1,
      cellClass: styles['centered-cell'],
    },
    {
      headerName: '인수상태',
      field: 'accSts',
      cellClass: styles['centered-cell'],
      cellRenderer: params => {
        switch(params.data.accSts) {
          case 'TA0401':
            if (params.data.tfrSts === 'TA0301') {
              return <span>-</span>;
            } else {
              return <Button onClick={ () => handleMoveToGasAccessionArchive(params.data) } size='small' icon={ filePenIcon() }>인수확인</Button>;
            }
          case 'TA0402':
            if (params.data.totalArch !== params.data.chkArch) {
              return <Button onClick={ () => handleMoveToGasAccessionArchive(params.data) } size='small' icon={ filePenIcon() }>인수확인</Button>;
            } else {
              return <Button onClick={ () => onClickConfirmationBtn(params.data) } size='small' icon={ filePenIcon() }>인수확정</Button>;
            }
          case 'TA0403':
            return <span>인수완료</span>;
          default:
            return <span>-</span>;
        }
      },
      width: '125px',
    },
    {
      headerName: '수량',
      field: 'checkCnt',
      cellClass: styles['centered-cell'],
      cellRenderer: cellParam => {
        const totCnt = cellParam.data.archQuantity;
        const chkCnt = cellParam.data.chkArch;
        return (
          <span>
            { `${ totCnt } / ${ chkCnt }` }
          </span>
        );
      },
      width: '110px',
    }
    ,
    {
      headerName: '목록 다운로드',
      cellClass: styles['centered-cell'],
      cellRenderer: cellParam => {
        if(cellParam.data.tfrSts === 'TA0303' && cellParam.data.accSts === 'TA0403') {
          return <Button onClick={ ()=> handleClickDownloadBtn() } icon={ fileTextIcon() } size={ 'small' } />;
        } else {
          return <span>-</span>;
        }
      },
      width: '80px',
    }
  ]);

  const [ confirmOpen, setConfirmOpen ] = useState(false);
  const [ validationErrors, setValidationErrors ] = useState([]);
  const [ formSearch ] = Form.useForm();

  const searchValueRef = useRef({
    ldgrNm: '',
    fundNm: '',
    tfrYear: '',
    accOrgId: '',
    tfrOrgId: '',
    accSts: '',
  });

  const paginationValueRef = useRef({
    total: 0,
    currentPage: 1,
    rowPerPage: 20,
  });

  const columns = 2;
  const labelCol = 16;
  const wrapperCol = 16;

  useEffect(() => {
    requestGasAccessionLedgerList();
    setSelectOptions();
  },[ activeTabKey ]);

  const setSelectOptions = async () => {
    setAccessionState(await getSelectOptions({ groupCode:'TA04',defaultLabel: '전체' }));
  };

  const requestGasAccessionLedgerList = async () => {
    console.log('requestGasAccessionLedgerList=======================>');
    try {
      let params = {
        ...searchValueRef.current,
        rowPerPage: paginationValueRef.current.rowPerPage,
        currentPage: paginationValueRef.current.currentPage,
      };
      const response = await fetchGet('v1/accession/gasAccessions', { params });
      if(hasValidValue(response.data)) {
        const gasAccessionLedgerList = response.data.list;
        if(gasAccessionLedgerList) {
          setGasAccessionLedgerList(gasAccessionLedgerList);
          paginationValueRef.current = {
            ...paginationValueRef.current,
            total: response.data.total,
            currentPage: response.data.currentPage,
            rowPerPage: response.data.rowPerPage
          };
        }
      }
    } catch (error) {
      console.log('Error:', error);
    }
  };

  const handleClickColumn = (record, dataIndex) => {
    console.log('record:' + record);
    dataIndex === 'fundNm' && console.log('흐므륵 상세보기');
    dataIndex === 'ldgrNm' && console.log('흐므륵 상세보기');
  };

  /**
  const handleConfirmOpenChage = (newOpen) => {
    if(!newOpen) {
      setConfirmOpen(newOpen);
      return;
    }

    if(selectedRowKey.length === 0) {
      messageApi.open({
        type: 'warning',
        message: 'Notice',
        description: '선택한 행이 없습니다.'
      });
    } else {
      setConfirmOpen(newOpen);
    }
  };
  */
  const handleTableChange = (page, pageSize) => {
    paginationValueRef.current = {
      ...paginationValueRef.current,
      currentPage: page,
      rowPerPage: pageSize
    };
    requestGasAccessionLedgerList();
  };

  const pagination = {
    current: paginationValueRef.current.currentPage,
    total: paginationValueRef.current.total,
    pageSize: paginationValueRef.current.rowPerPage,
    pageSizeOptions: [ 20, 50, 100 ],
    onChange: handleTableChange,
    showSizeChanger: true,
    showQuickJumper: true,
    showTotal: (total) => getTotalPageString(total),
  };

  const handleMoveToGasAccessionArchiveBtn = async(record) => {
    formSearch.resetFields();
    formSearch.setFields({ ldgrNm:'', fundNm:'', tfrYear:'', accOrgId:'', tfrOrgId:'', accSts: '' });
    searchValueRef.current = { ...formSearch.getFieldsValue() };
    paginationValueRef.current = { ...paginationValueRef.current, currentPage:1, pageSize: 20 };
    handleMoveToGasAccessionArchive(record);

  };

  const handleClickRefreshBtn = () => {
    formSearch.resetFields();
    formSearch.setFieldValue({ fundNm: '', ldgrNm: '', tfrYear:'', accOrgId: '', tfrOrgId: '', accSts: '' }); /// 작성필요
  };

  const handleClickSearchBtn = async () => {
    const formValues = formSearch.getFieldsValue();
    searchValueRef.current = { ...formValues };
    paginationValueRef.current = { ...paginationValueRef.current, currentPage: 1, pageSize: 20 };
    requestGasAccessionLedgerList();
  };

  // Organization Search
  const [ accessionrOrgModalOpen,setAccessionOrgModalOpen ]=useState(false);
  const [ transferOrgModalOpen,setTransferOrgModalOpen ]=useState(false);

  const handleClickMagnifierBtn= (type) => {
    console.log('XXX',type);
    if (type === 'TfrOrganization') {
      setTransferOrgModalOpen(true);
    } else if (type ==='AccOrganization') {
      setAccessionOrgModalOpen(true);
    }
  };

  const handleClickAccOrgModalOkBtn = (selectedOrgArr) => {
    const { orgNo, orgNm } = selectedOrgArr[0];
    formSearch.setFieldValue('accOrgNo', orgNo);
    formSearch.setFieldValue('accOrgNm', orgNm);
    setAccessionOrgModalOpen(false);
  };

  const handleClickAccOrgModalCancelBtn = () => {
    setAccessionOrgModalOpen(false);
  };

  const handleClickTfrOrgModalOkBtn = (selectedOrgArr) => {
    const { orgNo, orgNm } = selectedOrgArr[0];
    formSearch.setFieldValue('tfrOrgNo', orgNo);
    formSearch.setFieldValue('tfrOrgNm', orgNm);
    setTransferOrgModalOpen(false);
  };

  const handleClickTfrOrgModalCancelBtn = () => {
    setTransferOrgModalOpen(false);
  };


  const searchBtnGroup = [
    <Button
      key={ 'refresh' }
      className={ 'btn-second-btn' }
      icon={ refreshIcon() }
      onClick={ handleClickRefreshBtn }
    ></Button>,
    <Button
      key={ 'search' }
      type="primary"
      icon={ searchIcon() }
      onClick={ handleClickSearchBtn }
    ></Button>
  ];


  /**
  const callCreateMessageApi = response => {
    handleClickRefreshBtn();
    handleClickSearchBtn();
    if(response.status === 200) {
      requestGasAccessionLedgerList();
      messageApi.open({
        type: 'success',
        message: 'accession',
        description: '안녕!'
      });
    } else if(response.status === 400) {
      setValidationErrors(response.errors);
    } else {
      messageApi.open({
        type: 'warning',
        message: 'accession',
        description: '실패'
      });
    }
  };
  */

  return (
    <>
      { contextHolder }
      <ContentsWrapper>
        <FormBody
          formInst={ formSearch }
          formTitle="검색"
          labelWrap={ false }
          formWidth={ '100%' }
          formBtnGroup={ searchBtnGroup }
          formLayout='vertical'
        >
          <FormItem
            columns={ columns }
            itemName={ 'ldgrNm' }
            itemLabel={ '당스명' }
            labelCol={ labelCol }
            wrapperCol={ wrapperCol }
            initialValue={ '' }
          >
            <Input
              placeholder=''
            />

          </FormItem>
          <FormItem
            columns={ columns }
            itemName={ 'fundNm' }
            itemLabel={ '흐므륵명' }
            labelCol={ labelCol }
            wrapperCol={ wrapperCol }
            initialValue={ '' }
          >
            <Input
              placeholder=''
            />

          </FormItem>
          <FormItem
            columns={ columns }
            itemName={ 'tfrYear' }
            itemLabel={ '이관년도' }
            labelCol={ labelCol }
            wrapperCol={ wrapperCol }
            initialValue={ '' }
          >
            <Input
              placeholder=''
            />

          </FormItem>
          <FormItem
            columns={ columns }
            itemName={ 'accOrgNm' }
            itemLabel ={ 'Бүрдүүлэлт хүлээн авах төрийн архив 인수기관' }
            labelCol={ labelCol }
            wrapperCol={ wrapperCol }
          >
            <Input.Search
              placeholder="Organization Name"
              readOnly
              onSearch={ () => handleClickMagnifierBtn('AccOrganization') }
            />
          </FormItem>

          <FormItem
            columns={ columns }
            itemName={ 'tfrOrgNm' }
            itemLabel ={ 'Бүрдүүлэлт хийх байгууллагын нэр 이관기관' }
            labelCol={ labelCol }
            wrapperCol={ wrapperCol }
          >
            <Input.Search
              placeholder="Organization Name"
              readOnly
              onSearch={ () => handleClickMagnifierBtn('TfrOrganization') }
            />
          </FormItem>
          <FormItem
            columns={ columns }
            itemName={ 'accSts' }
            itemLabel ={ '인수상태' }
            labelCol={ labelCol }
            wrapperCol={ wrapperCol }
            initialValue={ '' }
          >
            <Select
              options={ accessionState }
            />
          </FormItem>
          <FormItem
            itemName="accOrgNo"
            labelCol={ 0 }
            wrapperCol={ 0 }
          >
          </FormItem>
          <FormItem
            itemName="tfrOrgNo"
            labelCol={ 0 }
            wrapperCol={ 0 }
          >
          </FormItem>
        </FormBody>
      </ContentsWrapper>
      <NotifyOrganizationSearchModal
        isModalOpen={ accessionrOrgModalOpen }
        modalOkBtn={ handleClickAccOrgModalOkBtn }
        modalCancelBtn={ handleClickAccOrgModalCancelBtn }
      />

      { /* <OrganizationSearchModal
        isModalOpen={ transferOrgModalOpen }
        modalOkBtn={ handleClickTfrOrgModalOkBtn }
        modalCancelBtn={ handleClickTfrOrgModalCancelBtn }
      /> */ }
      <NotifyOrganizationSearchModal
        isModalOpen={ transferOrgModalOpen }
        modalOkBtn={ handleClickTfrOrgModalOkBtn }
        modalCancelBtn={ handleClickTfrOrgModalCancelBtn }
      />
      <div>
        <>
          <ContentsWrapper
            cardHeadTitle={ '인수관리' }
            cardHeadBtn={ [
              <GridButton key={ 'grid_btn' } openDrawer={ handleClickOpenDrawer } />
            ] }
            cardAction={ null }>
            <GridWrapper>
              <DataGrid
                gridId='ag_grid_gas_accession_ledger'
                gridRef={ gasAccessionLedgerGridRef }
                rowData={ gasAccessionLedgerList }
                domLayout={ 'autoHeight' }
                columnDefs={ gasAccessionLedgerColumnDefs }
                gridOptions={ gridOptions }
                rowSelection={ 'multiple' }
                drawer={ drawer }
                closeDrawer={ handleClickCloseDrawer }
              />
            </GridWrapper>
          </ContentsWrapper>
          <div className={ 'div-pagination-box' }>
            <Pagination { ...pagination } />
          </div>
        </>

      </div>
    </>
  );


}
